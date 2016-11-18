# JuliaWebAPI.jl

[![Build Status](https://travis-ci.org/JuliaWeb/JuliaWebAPI.jl.svg?branch=master)](https://travis-ci.org/JuliaWeb/JuliaWebAPI.jl)
[![Coverage Status](https://coveralls.io/repos/github/JuliaWeb/JuliaWebAPI.jl/badge.svg?branch=master)](https://coveralls.io/github/JuliaWeb/JuliaWebAPI.jl?branch=master)

Facilitates wrapping Julia functions into a remote callable API via message queues (e.g. ZMQ) and HTTP.

It can plug in to a different messaging infrastructure through an implementation of transport (`AbstractTransport`) and message format (`AbstractMsgFormat`).
Multiple instances of the front (HTTP API) and back (Julia methods) end can help scale an application.

Combined with [JuliaBox](https://juliabox.org/), it helps deploy Julia packages and code snippets as publicly hosted, auto-scaling HTTP APIs.

## Example Usage

Create a file `srvr.jl` with the following code

```julia
# Load required packages
using JuliaWebAPI
using Compat

# Define functions testfn1 and testfn2 that we shall expose
function testfn1(arg1, arg2; narg1=1, narg2=2)
    return (parse(Int, arg1) * parse(Int, narg1)) + (parse(Int, arg2) * parse(Int, narg2))
end

testfn2(arg1, arg2; narg1=1, narg2=2) = testfn1(arg1, arg2; narg1=narg1, narg2=narg2)

# Expose testfn1 and testfn2 via a ZMQ listener
process([(testfn1, true), (testfn2, false)], "tcp://127.0.0.1:9999"; bind=true)
```

Start the server process in the background. This process will run the ZMQ listener.
````
julia srvr.jl &
````

Then, on a Julia REPL, run the following code
```julia
using JuliaWebAPI   #Load package
using Logging
Logging.configure(level=INFO);

#Create the ZMQ client that talks to the ZMQ listener above
const apiclnt = APIInvoker("tcp://127.0.0.1:9999");

#Starts the HTTP server in current process
run_rest(apiclnt, 8888)
```

Then, on your browser, navigate to `http://localhost:8888/testfn1/4/5?narg1=6&narg2=4`

This will return the following JSON response to your browser, which is the result of running the `testfn1` function defined above:
`{"data"=>44,"code"=>0}`


## JuliaBox Deployment

Deploying on JuliaBox takes care of most of the boilerplate code. To expose a simple fibonacci generator on JuliaBox, paste the following 
code as the API command:
````
fib(n::AbstractString) = fib(parse(Int, n));
fib(n::Int) = (n < 2) ? n : (fib(n-1) + fib(n-2));
process([(fib, false)]);
````

Notice that we need to specify a lot less detail on JuliaBox. JuliaBox connects the API servers to a queue, instead of the server having to listen 
for requests. The obvious packages are aleady imported.

The JuliaBox API command must however be concisely expressed within 512 bytes without new lines. To run larger applications, simply package up the 
code as a Julia package and install the package as part of the command. For an example, see the [Juliaset API package](https://github.com/tanmaykm/Juliaset.jl).
