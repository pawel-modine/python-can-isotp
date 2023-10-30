.. _Socket:

IsoTp Sockets
=============

Introduction
------------

Under Linux, CAN interfaces can be managed the same way as network interfaces. The support for each CAN driver is written directly in the Linux kernel, inside a module called SocketCAN. Therefore, it is possible to send data over CAN through sockets, which offer a unique interface for all CAN drivers.

SocketCAN allows a user the send CAN messages on the CAN layer and, starting from Linux 5.10.0, ISO-TP frames as well. Meaning that a user can write a payload into an ISO-TP socket, this payload will be subdivided into multiple CAN messages and sent to a receiver following the ISO-15765 standard

For version prior to 5.10.0, this `out-of-tree loadable kernel module <https://github.com/hartkopp/can-isotp>`_  can be compiled and loaded in the Linux kernel, enabling ISO-TP sockets.

A socket is a standard interface for communication protocols and as anything generic, it can be complex to configure. The ``isotp.socket``  is a wrapper over a native IsoTP socket providing a friendly and pythonic interface for easy configuration. It does not offer any additional functionality that your operating system can't provide, it makes the syntax clean and warns you in case of wrong usage.


Troubleshooting
---------------

 - **My socket module does not include the `CAN_ISOTP` constant**
 
That means that your Python version does not include support for IsoTP protocol. It should be included starting from Python 3.7, under Linux build only. See `Python issue <https://bugs.python.org/issue30987>`_ and `Pull request <https://github.com/python/cpython/pull/2956>`_

 - **When I create the socket, I get `OSError [errno XX] : Protocol not supported`.**
 
Your Linux kernel does not support ISO-TP sockets, you need to manually load it. 
Follow the steps given it the `out-of-tree repository <https://github.com/hartkopp/can-isotp>`_. You needs to compile the module, install the `.ko` file and then run `insmod can-isotp.ko` as Super User. Then your OS will accept to create an ISO-TP sockets.

 - **I get a timeout when calling send().**
 
A normal use case for that issue is that the receiver failed to respond with a flow control message.

Also, some kernel versions have been hit with `a bug <https://lore.kernel.org/linux-can/04fd32bc-b1c4-b9c3-3f8b-7987704a1f85@hartkopp.net/T/#m9615a4ecbdb742749886af73414e476498c93d51>`_. 
A race condition within the isotp kernel module can cause the send() function to return an unexpected timeout. The patch has been published in `this PR <https://lore.kernel.org/linux-can/96d31e8c-fa26-4e32-4c36-768981f20a54@hartkopp.net/T/#u>`_.
This `github issue <https://github.com/pylessard/python-udsoncan/issues/178>`_ also covers the matter

Examples
--------

Without this project
####################

.. code-block:: python

   SOL_CAN_ISOTP = 106 # These constants exist in the module header, not in Python.
   CAN_ISOTP_RECV_FC = 2
   # Many more exists.

   import socket
   import struct

   s = socket.socket(socket.AF_CAN, socket.SOCK_DGRAM, socket.CAN_ISOTP)
   s2 = socket.socket(socket.AF_CAN, socket.SOCK_DGRAM, socket.CAN_ISOTP)
   # Configuring the sockets with ugly struct.pack() that requires knowledge of the driver
   s.setsockopt(SOL_CAN_ISOTP, CAN_ISOTP_RECV_FC, struct.pack("=BBB", 0x10, 3,0)) #bs, stmin, wftmax
   #s.setsockopt(SOL_CAN_ISOTP, CAN_ISOTP_OPTS, struct.pack(...))
   #s.setsockopt(SOL_CAN_ISOTP, CAN_ISOTP_LL_OPTS, struct.pack(...))

   s.bind(("vcan0", 0x123, 0x456)) #rxid, txid with confusing order.
   s2.bind(("vcan0", 0x456, 0x123)) #rxid, txid
   s2.send(b"Hello, this is a long payload sent in small chunks of 8 bytes.")
   print(s.recv(4095))

With this project
#################

.. code-block:: python

   import isotp

   s = isotp.socket()
   s2 = isotp.socket()
   # Configuring the sockets.
   s.set_fc_opts(stmin=5, bs=10)
   #s.set_general_opts(...)
   #s.set_ll_opts(...)

   s.bind("vcan0", isotp.Address(rxid=0x123, txid=0x456)) 
   s2.bind("vcan0", isotp.Address(rxid=0x456, txid=0x123))
   s2.send(b"Hello, this is a long payload sent in small chunks of 8 bytes.")
   print(s.recv()) 


Usage
-----

.. autoclass:: isotp.socket

.. automethod:: isotp.socket.bind(interface, address)

-----

To configure a socket, few methods are available 

.. py:method:: socket.set_opts(optflag=None, frame_txtime=None, ext_address=None, txpad=None, rxpad=None, rx_ext_address=None, tx_stmin=None)
   
   Sets the general options of the socket

   :param optflag: A list of flags modifying the protocol behaviour. Refer to :class:`socket.flags<isotp.socket.flags>`
   :type optflag: int

   :param frame_txtime: Frame transmission time (N_As/N_Ar) in nanoseconds.
   :type frame_txtime: int

   :param ext_address: The extended address to use. If not None, flags.EXTEND_ADDR will be set.
   :type ext_address: int

   :param txpad: The byte to use to pad the transmitted CAN messages. If not None, flags.TX_PADDING will be set
   :type txpad: int

   :param rxpad: The byte to use to pad the transmitted CAN messages. If not None, flags.RX_PADDING will be set
   :type rxpad: int

   :param rx_ext_address: The extended address to use in reception. If not None, flags.RX_EXT_ADDR will be set
   :type rx_ext_address: int

   :param tx_stmin: Sets the transmit separation time (time between consecutive frame) in nanoseconds. This value will override the value received through FlowControl frame. If not None, flags.FORCE_TXSTMIN will be set
   :type tx_stmin: int


.. py:method:: socket.set_fc_opts(bs=None, stmin=None, wftmax=None)
   
   Sets the flow control options of the socket

   :param bs: The block size sent in the flow control message. Indicates the number of consecutive frame a sender can send before the socket sends a new flow control. A block size of 0 means that no additional flow control message will be sent (block size of infinity)
   :type bs: int

   :param stmin: The minimum separation time sent in the flow control message. Indicates the amount of time to wait between 2 consecutive frame. This value will be sent as is over CAN. Values from 1 to 127 means milliseconds. Values from 0xF1 to 0xF9 means 100us to 900us. 0 Means no timing requirements
   :type stmin: int

   :param wftmax: Maximum number of wait frame (flow control message with flow status=1) allowed before dropping a message. 0 means that wait frame are not allowed
   :type wftmax: int

.. py:method:: socket.set_ll_opts(mtu=None, tx_dl=None, tx_flags=None)
   
   Sets the link layer options. Default values are set to work with CAN 2.0. Link layer may be configure to work in CAN FD.

   :param mtu: The internal CAN frame structure size. Possible values are defined in :class:`isotp.socket.LinkLayerProtocol<isotp.socket.LinkLayerProtocol>`
   :type mtu: int

   :param tx_dl: The CAN message payload length. For CAN 2.0, this value should be 8. For CAN FD, possible values are 8,12,16,20,24,32,48,64
   :type tx_dl: int

   :param tx_flags: Link layer flags.
   :type tx_flags: int

-----

.. autoclass:: isotp.socket.flags
   :members:
   :undoc-members:
   :member-order: bysource

.. autoclass:: isotp.socket.LinkLayerProtocol
   :members:
   :undoc-members:
   :member-order: bysource
