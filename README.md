# Getcas

A protocol for getting byte strings from a **c**ontent-**a**ddressable **s**torage.

**Status: freshly specified, needs more time before it can be declared stable.**

## What, Why, and How?

[Content-addressable storage](https://en.wikipedia.org/wiki/Content-addressable_storage) allows retrieval of byte strings by specifying a short name. The name of each different string is simply a function of the string itself, typically a [cryptographical hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function). A minimal protocol allowing a client to read from a CAS held by a server consists of the client sending names and the server answering with the corresponding strings.

This naive approach can become problematic in the case of network failures, because partially transmitted strings would have to be fully transmitted again if the connection is lost. If the size of a string is too large compared to the network bandwidth and expected failure rate, the probability that the string can ever be retrieved becomes too small. Getcas thus allows the client to specify an offset into the string at which transmission should begin.

If the client uses this offset feature because it already has a prefix of the string, but this prefix is actually incorrect, this would lead to reconstruction of an incorrect string. This would only be detected at the end of a successful transmission. This is protected against by having the client also transmit the name of the prefix already has.

Finally, the protocol provides flow control by building on the [reqres](https://github.com/AljoschaMeyer/reqres) protocol.

## Protocol Concepts

The protocol is a request-response protocol with static request and streaming responses.

A request consists of the name of the requested string, and optionally a nonzero offset `o` and the expected name of the length-`o` prefix of the requested string.

We now look at the response to a request for name `n` at offset `o` (we define `o := 0` if no offset was specified) and with the expected prefix name `p`. The first item of the response indicates whether the response will transmit the string suffix starting at `o` (if `p` matches), or whether it will transmit the full string (if `p` does not match). The repeated items of a response are the individual bytes of the string. The last item of the responses the unit type, simply indicating the end of the transmission without containing any further information.

If the server does not have the requested string, it sends a first item indicating that it will transmit the full strength, followed immediately by the last item.

If the server sends the last item, but the overall transmittal data does not hash to the expected name, there are a couple of different interpretations for the client: the server might have fed it garbage deliberately, the server might have only had a prefix of the requested string, or the server might not have checked the name of the prefix and the prefix originally stored at the client was garbage. The protocol does not specify which interpretation the client should favor or how it should react.

## Encoding

The protocol is an instantiation of [reqres](https://github.com/AljoschaMeyer/reqres) with static request and streaming responses.

A request is encoded by encoding the requested name, followed by a [VarU64](https://github.com/AljoschaMeyer/varu64). If the integer is zero, no offset and prefixed name specified, and the request encoding ends. Otherwise, the integer encodes the offset at which to start the response, the integer is then followed by the encoding of the expected prefix name.

If an offset was specified, the first response item is encoded as the byte `0x00` if the response starts at the specified offset, or as the byte `0x01` if the response transmits the full string. If no offset was specified, the first response item does not matter and is thus encoded as the empty string.

The repeated response items are the bytes of the string, they are encoded as themselves.

The last response item is encoded as the empty string, because it carries no information.
