#FazCAN MPC2515 CAN Driver Documentation
The FazCAN driver is a C/C++/Arduino driver for the MCP2515 CAN controller
module. It allows the user to create a two-wire communication network between
two or more CAN modules or to interface with other CAN networks.

To use the driver in the Arduino build environment, copy the CAN directory into
the "libraries" subdirectory of your Arduino sketchbook directory.

To use the driver in a normal C/C++ build environment, open "my_spi.h" and
delete the line that reads:

```c++
#include <Arduino.h>
```

For straight C code, rename "mcp2515.cpp" to "mcp2515.c." Both C and C++ will
need to directly interface with mcp2515.c, as the C++ interface is currently
dependent on the Arduino environment. The rest of this documentation describes
using the Arduino interface.

To use the CAN module, you will need to include both the SPI.h and CAN.h
headers in your sketch:

```c++
#include <SPI.h>
#include <CAN.h>
```

Before using the CAN module, you must call `CAN.begin()`, specifying the bus
speed as an argument. Several standard speeds are defined in CAN.h, such as
`CAN_SPEED_125000`, which indicates a speed for 125000 bits per second:

```c++
CAN.begin (CAN_SPEED_125000);
```

Alternatively, you can directly specify the length of each bit (called the bit
period) in nanoseconds. If you want a bit period of 25 microseconds
(equivalent to a bus speed of 40000 bits per second), you can specify it this
way:

```c++
CAN.begin (25000);
```
Next you must set the CAN mode. The modes are defined in CAN.h. You will
probably want to use `CAN_MODE_NORMAL` or `CAN_MODE_LISTEN_ONLY`.

```c++
CAN.setMode (CAN_MODE_NORMAL);
```

`CAN_MODE_NORMAL` will allow the MCP2515 to transmit and receive on the CAN bus
normally. `CAN_MODE_LISTEN_ONLY` only allows the MCP2515 to receive, preventing
it from interacting on the bus. This ensures that the module does not interfere
with regular network activity. (A CAN network requires at least two
transmitting CAN nodes to function correctly. If you are creating your own
network, you cannot have only one NORMAL and one LISTEN node, even if the
NORMAL node is the only node sending messages. This is due to a technical
detail of the CAN communication protocol. As long as there are at least two
NORMAL nodes, everything can work fine.)

To determine if you have received a CanMessage, call `CAN.available ()`. If
this function returns true, there is a message to be retrieved. Retrieve the
message by calling `CAN.getMessage ()`. You must have declared a CanMessage
variable to receive the message into.

```c++
CanMessage message;
if (CAN.available ()) {
    message = CAN.getMessage ();
}
```

A CanMessage has an identifier, which is a number that indicates what kind of
data that message has. There is no universal set of CAN identifiers; instead
the identifier is specific to the application. If you are creating your own CAN
network, you can pick the CAN identifiers. If you are interfacing with another
network, you need to use the identifiers defined by that network protocol.

The CAN identifier can be either 11 bits or 29 bits. If the CAN message uses an
11-bit identifier, the "extended" flag of the CAN message will be false. If the
message uses a 29-bit identifier, the "extended" flag will be true. The CAN ID
and extended flags can be written and read directly from the message variable:

```c++
id = message.id;
if (message.extended) {
    // Handle extended message
} else {
    // Handle non-extended message
}
```

You only need to think about the extended flag if you are interfacing with an
existing CAN network. If you are creating your own network, you can ignore this
and use identifier numbers that are less than 2048 (hex: 0x7FF).

A CanMessage also has from zero to eight data bytes. The message identifier
indicates what type of data is included in the message. If you are creating a
CAN network of sensors that detect temperature and GPS position, you might use
one identifier for a CanMessage whose data contains the temperature, and
another identifier for a CanMessage whose data contains the GPS coordinates.
When a receiver receives a message, it can use the identifier to determine how
to use the data in the message. The extended flag does not change anything
about the message data. Both extended and non-extended messages can have zero
to eight bytes of data.

The data can be retrieved from the CanMessage in two ways. The first is to read
the bytes of data directly from the message array:

```c++
byte0 = message.data[0];
byte1 = message.data[1];
```

The length field tells you how many bytes of data are in the message:

```c++
for (i = 0; i < message.len; i++) {
    // Process data bytes
}
```

The second way is specific to this library, and may not work when interfacing
with existing CAN networks. The functions `getByteFromData`, `getIntFromData`,
and `getLongFromData` allow you to read one variable from the message. The
reading begins at the beginning of the data array, and each read moves the
reading position further along the message data. To correctly retrieve the
data, you must read out the variables in the order they were set. Use the
correct function for the length of the data you are trying to read:

```c++
long time;
byte pulses;
int temperature;
time = message.getLongFromData ();
pulses = message.getByteFromData ();
temperature = message.getIntFromData ();
```

Sending messages is very similar to receiving messages. Declare a CanMessage
variable and set the identifier. (If you are using extended messages, remember
to set the extended flag too).

```c++
CanMessage message;
message.id = 0x501;
```

Next set the data. You can do this in two ways. The first is by writing
directly into the data array:

```c++
message.data[0] = my_val1;
message.data[1] = my_val2;
message.len = 2;
```

Note that the data array is an array of bytes, so you cannot store an int or
long type in one array element:

```c++
int my_val = 12345;
message.data[0] = my_val;  // WRONG
```

The second way is to use the set functions:

```c++
message.clear ();                   // Clears all previous data,
                                    //     8 bytes available
message.setLongData (time);         // Uses 4 bytes, 4 left
message.setByteData (pulses);       // Uses 1 byte, 3 left
message.setIntData (temperature);   // Uses 2 bytes, 1 left
```

The set functions store a variable in the data array and set the data length
for you. You can use more than one set function to fill the message data with
more than one variable. Remember that the message is only 8 bytes, so you
cannot set more than two longs, four ints, eight bytes, or a combination adding
up to eight. (See Ardunio documentation: byte, int, long)

After setting the identifier and data, a message is ready to be sent. You must
wait for the CAN module to be ready before sending. When it is ready, the
message can be sent by calling the send method:

```c++
while (CAN.ready () == false) {
    // Wait for CAN to be ready
}

message.send ();
```

A CanMessage variable can be reused. After sending you can simply call send
again to send the same message again. You can also clear the message by calling
the clear function, then use the set functions to create a different message.

For more information, see the examples included in the FazCAN library.
