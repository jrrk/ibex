.. _load-store-unit:

Load-Store Unit
===============

The Load-Store Unit (LSU) of the core takes care of accessing the data memory.
Loads and stores of words (32 bit), half words (16 bit) and bytes (8 bit) are supported.

Signals that are used by the LSU:

+-------------------------+-----------+-----------------------------------------------+
| Signal                  | Direction | Description                                   |
+=========================+===========+===============================================+
| ``data_req_o``          | output    | Request valid, must stay high until           |
|                         |           | ``data_gnt_i`` is high for one cycle          |
+-------------------------+-----------+-----------------------------------------------+
| ``data_addr_o[31:0]``   | output    | Address, word aligned                         |
+-------------------------+-----------+-----------------------------------------------+
| ``data_we_o``           | output    | Write Enable, high for writes, low for        |
|                         |           | reads. Sent together with ``data_req_o``      |
+-------------------------+-----------+-----------------------------------------------+
| ``data_be_o[3:0]``      | output    | Byte Enable. Is set for the bytes to          |
|                         |           | write/read, sent together with ``data_req_o`` |
+-------------------------+-----------+-----------------------------------------------+
| ``data_wdata_o[31:0]``  | output    | Data to be written to memory, sent together   |
|                         |           | with ``data_req_o``                           |
+-------------------------+-----------+-----------------------------------------------+
| ``data_gnt_i``          | input     | The other side accepted the request.          |
|                         |           | Outputs may change in the next cycle.         |
+-------------------------+-----------+-----------------------------------------------+
| ``data_rvalid_i``       | input     | ``data_err_i`` and ``data_rdata_i`` hold      |
|                         |           | valid data when ``data_rvalid_i`` is high.    |
|                         |           | This signal will be high for exactly one      |
|                         |           | cycle per request.                            |
+-------------------------+-----------+-----------------------------------------------+
| ``data_err_i``          | input     | Error response from the bus or the memory:    |
|                         |           | request cannot be handled. High in case of an |
|                         |           | error.                                        |
+-------------------------+-----------+-----------------------------------------------+
| ``data_rdata_i[31:0]``  | input     | Data read from memory                         |
+-------------------------+-----------+-----------------------------------------------+


Misaligned Accesses
-------------------

The LSU is able to handle misaligned memory accesses, meaning accesses that are not aligned on natural word boundaries.
However, it does so by performing two separate word-aligned accesses.
This means that at least two cycles are needed for misaligned loads and stores.

.. _lsu-protocol:

Protocol
--------

The protocol that is used by the LSU to communicate with a memory works as follows:

1. The LSU provides a valid address in ``data_addr_o`` and sets ``data_req_o`` high. In the case of a store, the LSU also sets ``data_we_o`` high and configures ``data_be_o`` and ``data_wdata_o``. The memory then answers with a ``data_gnt_i`` set high as soon as it is ready to serve the request. This may happen in the same cycle as the request was sent or any number of cycles later.

2. After receiving a grant, the address may be changed in the next cycle by the LSU. In addition, the ``data_wdata_o``, ``data_we_o`` and ``data_be_o`` signals may be changed as it is assumed that the memory has already processed and stored that information.

3. The memory answers with a ``data_rvalid_i`` set high for exactly one cycle to signal the response from the bus or the memory using ``data_err_i`` and ``data_rdata_i`` (during the very same cycle). This may happen one or more cycles after the grant has been received. If ``data_err_i`` is low, the request could successfully be handled at the destination and in the case of a load, ``data_rdata_i`` contains valid data. If ``data_err_i`` is high, an error occured in the memory system and the core will raise an exception.

:numref:`timing1`, :numref:`timing2` and :numref:`timing3` show example-timing diagrams of the protocol.

.. wavedrom::
   :name: timing1
   :caption: Basic Memory Transaction

   {"signal":
     [
       {"name": "clk", "wave": "p......"},
       {"name": "data_req_o", "wave": "01.0..."},
       {"name": "data_addr_o", "wave": "x=.xxxx", "data": ["Address"]},
       {"name": "data_we_o", "wave": "x=.xxxx", "data": ["WE"]},
       {"name": "data_be_o", "wave": "x=.xxxx", "data": ["BE"]},
       {"name": "data_wdata_o", "wave": "x=.xxxx", "data": ["WData"]},
       {"name": "data_gnt_i", "wave": "0.10..."}, 
       {"name": "data_rvalid_i", "wave": "0..10.."},
       {"name": "data_err_i", "wave": "xxx=xxx", "data": ["Err"]},
       {"name": "data_rdata_i", "wave": "xxx=xxx", "data": ["RData"]}

     ],
     "config": { "hscale": 2 }
    }

.. wavedrom::
   :name: timing2
   :caption: Back-to-back Memory Transaction

   {"signal":
     [
       {"name": "clk", "wave": "p......"},
       {"name": "data_req_o", "wave": "01.0..."},
       {"name": "data_addr_o", "wave": "x==xxxx", "data": ["Addr1", "Addr2"]},
       {"name": "data_we_o", "wave": "x==xxxx", "data": ["WE1", "WE2"]},
       {"name": "data_be_o", "wave": "x==xxxx", "data": ["BE1", "BE2"]},
       {"name": "data_wdata_o", "wave": "x==xxxx", "data": ["WData1", "Wdata2"]},
       {"name": "data_gnt_i", "wave": "01.0..."},
       {"name": "data_rvalid_i", "wave": "0.1.0.."},
       {"name": "data_err_i", "wave": "xx==xxx", "data": ["Err1", "Err2"]},
       {"name": "data_rdata_i", "wave": "xx==xxx", "data": ["RData1", "RData2"]}
     ],
     "config": { "hscale": 2 }
   }
   
.. wavedrom::
   :name: timing3
   :caption: Slow Response Memory Transaction

   {"signal":
     [
       {"name": "clk", "wave": "p......"},
       {"name": "data_req_o", "wave": "01..0.."},
       {"name": "data_addr_o", "wave": "x=..xxx", "data": ["Address"]},
       {"name": "data_we_o", "wave": "x=..xxx", "data": ["WE"]},
       {"name": "data_be_o", "wave": "x=..xxx", "data": ["BE"]},
       {"name": "data_wdata_o", "wave": "x=..xxx", "data": ["WData"]},
       {"name": "data_gnt_i", "wave": "0..10.."}, 
       {"name": "data_rvalid_i", "wave": "0....10"},
       {"name": "data_err_i", "wave": "xxxxx=x", "data": ["Err"]},
       {"name": "data_rdata_i", "wave": "xxxxx=x", "data": ["RData"]}
     ],
     "config": { "hscale": 2 }
   }
