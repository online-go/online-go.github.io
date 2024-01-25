---
layout: default
---

# Online-Go.com Developer Resources

## gtp2ogs project and documentation

To connect a bot to the online-go.com servers, you'll need to use the [gtp2ogs](https://github.com/online-go/gtp2ogs)
project which will allow you to connect a bot that speaks the [Go Text Protocol](https://www.gnu.org/software/gnugo/gnugo_19.html)
to the online-go.com servers.

## Realtime API

Playing games, chatting, accessing the AI review system, and many more features
relies on establishing a persistent WebSocket connection to the server and
exchanging messages.

The protocol is documented in the Goban project here:

[GobanSocket Realtime API documentation](/goban/modules/protocol.html)

## REST API

Creating challenges, adding friends, joining tournaments and ladders, loading tsumego,
and many more features are done via REST calls to the REST API.

The REST API documentation is a work in progress.
