.. _at_commands:

AT commands
###########

The Modem library offers the nrf_modem_at APIs to exchange AT commands with the modem and receive AT notifications.

The AT interface has two functions which cover the most common use cases, :c:func:`nrf_modem_at_printf` and :c:func:`nrf_modem_at_scanf`.

:c:func:`nrf_modem_at_printf` can be used to send a formatted AT command to the modem when only interested in the generic result of the operation, e.g. "OK", "ERROR", "CME ERROR" or "CMS ERROR".
This function works similarly to `printf` from the standard C library, and it supports all the format specifiers supported by the `printf` implementation of the selected C library.
The following snippet shows how to use :c:func:`nrf_modem_at_printf` to send a formatted AT command to the modem and check the result of the operation.

.. code-block:: c

  int cfun_control(int mode)
  {
	  int err;

	  err = nrf_modem_at_printf("AT+CFUN=%d", mode);
	  switch (err) {
	  case 0: break; /* Modem response is "OK" */
	  case 1: break; /* Modem response is "ERROR" */
	  case 2: break; /* Modem response is "CME ERROR" */
	  case 3: break; /* Modem response is "CMS ERROR" */
	  }

	  return err;
  }

  int foo(void)
  {
	  /* Send AT+CFUN=1 */
	  cfun_control(1);
	  /* Send AT+CFUN=4 */
	  cfun_control(4);
  }

:c:func:`nrf_modem_at_scanf` can be used to send an AT command to the modem and parse the response according to a specified format.
This function works similarly to `scanf` from the standard C library, and it supports all the format specifiers supported by the `scanf` implementation of the selected C library.
This function can be used when querying the modem for data.
The following snippet shows how to use :c:func:`nrf_modem_at_scanf` to read the modem network registration status via `AT+CEREG?`.

.. code-block:: c

  void cereg_read(void)
  {
	  int rc;
	  int status;

	  /* The `*` sub-specifier discards the result of the match.
	   * The data is read but it is not stored in the location pointed by an argument.
	   */
	  rc = nrf_modem_at_scanf("AT+CEREG?", "+CEREG: %*d,%d", &status);

	  /* Upon returning, `rc` contains the number of matches */
	  if (rc > 1) {
		  /* We have matched one argument */
		  printk("Network registration status: %d\n", status);
	  } else {
		  /* No arguments where matched */
	  }
  }

The remaining two functions cover more advanced use-cases, they are :c:func:`nrf_modem_at_cmd` and its asynchronous version, :c:func:`nrf_modem_at_cmd_async`.
These functions allow sending formatted AT commands to the modem, and receive the modem response into a user-supplied buffer.
The user can then parse the buffer as necessary, for example by using the C library function :c:func:`sscanf`, thus achieving the functionality of :c:func:`nrf_modem_at_printf` and :c:func:`nrf_modem_at_scanf` combined.

AT Notifications
################

When the AT interface is initialized, the user provide a callback to receive AT notifications.
The callback is invoked in an interrupt context. The user is responsible for rescheduling processing of AT notifications as appropriate.
In NCS, this is taken care of by the `AT Monitor` library and the Integration layer.


Socket interface
################

@note: The socket-based AT interface is deprecated since version v1.3.0. Use the nrf_modem_at interface instead.

You can use the socket interface to send AT commands to the LTE modem and to receive the responses.

To do so, create a socket with the proprietary address family :c:type:`NRF_AF_LTE` and the protocol family :c:type:`NRF_PROTO_AT`.
You can then write AT command strings to that socket using :c:func:`nrf_send` and :c:func:`nrf_write`, and they are sent to the modem.
The responses are available when you read from the socket using :c:func:`nrf_read` and :c:func:`nrf_recv`.
Received AT messages are not truncated; to read the full response, provide a sufficiently large buffer or fetch the full response in several read operations.

See the `AT Commands Reference Guide`_ for detailed information on the available AT commands.

The following BSD socket functions are available for the :c:type:`NRF_PROTO_AT` protocol family:

* :c:func:`nrf_socket` for creating a socket
* :c:func:`nrf_close` for closing a socket
* :c:func:`nrf_fcntl` for managing socket options
* :c:func:`nrf_read` for reading data from a socket
* :c:func:`nrf_recv` for reading data from a socket and concatenating several received messages into one receive buffer
* :c:func:`nrf_write` for writing data to a socket
* :c:func:`nrf_send` for writing data to a socket using specific flags

By default, read and write functions are blocking.
To make them non-blocking, use :c:func:`nrf_fcntl` to set the NRF_O_NONBLOCK flag, or pass NRF_MSG_DONTWAIT as flag to the function call.

The following code example shows how to send an AT command and receive the response:

.. code-block:: c

   #include "nrf_socket.h" // socket(), NRF_AF_LTE family type, NRF_PROTO_AT protocol.
   #include <string.h>     // strncmp()

   int func(void)
   {
       // Create a socket for AT commands.
       int fd = nrf_socket(NRF_AF_LTE, 0, NRF_PROTO_AT);

       // Write the AT command.
       nrf_write(fd, "AT+CEREG=2", 10);

       // Allocate a response buffer.
       char ok_buffer[10];

       // Read an AT message (read 10 bytes to ensure that the
       // entire message is consumed).
       int num_of_bytes_recvd = nrf_read(fd, ok_buffer, 10);

       // Compare buffer content against expected return value.
       if (strncmp("OK", ok_buffer, 2) != 0)
       {
           // Return in case of failure.
           return -1;
       }

       // Return on success.
       return 0;
   }
