# Java-NIO-Service

I want to build a simple restful service about Java NIO and Asynchronous programming.

The flow is below:
web browser ---> ajax --> server (service --> using socket to communicate with one standalone app that take 1 minutes to return)

server --> socket --> app (sleep one minute)

.... async programming....

server <--socket <-- app (when server receiving msg, invoke callback to process)

web browser <-- response (json) <--server

