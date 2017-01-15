# go-casper [![Travis](https://img.shields.io/travis/tcnksm/go-casper.svg?style=flat-square)][travis] [![MIT License](http://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)][license] [![Go Documentation](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)][godocs]

[travis]: https://travis-ci.org/tcnksm/go-casper
[license]: https://github.com/tcnksm/go-casper/blob/master/LICENSE
[godocs]: http://godoc.org/github.com/tcnksm/go-casper

Package `go-casper` is golang implementation of [H2O](https://github.com/h2o/h2o)'s [CASPer](https://h2o.examp1e.net/configure/http2_directives.html#http2-casper) (cache-aware server-push).

[Go 1.8](https://tip.golang.org/doc/go1.8) is going to support HTTP/2 server push. Server push allows us to send resources like CSS or JavaScript files before the client asks (so we can expect faster page rendering). As described on [this post](http://blog.kazuhooku.com/2015/10/performance-of-http2-push-and-server.html) or [this issue](https://github.com/h2o/h2o/issues/421), one of the important things to use server push is to know *when to push*. Since it's waste of the network bandwidth (and cause negative effects on response time), you should avoid to push the asset which has already been cached by the client. 

To solve these problem, [H2O](https://github.com/h2o/h2o), a server that provides full advantage of HTTP/2 features, introduces [CASPer](https://h2o.examp1e.net/configure/http2_directives.html#http2-casper). CASPer maintains a fingerprint of the browser caches ([Golomb-compressed](https://en.wikipedia.org/wiki/Golomb_coding) bloom filter) as a cookie, and cancels server-push if the fingerprint indicates the client is known to be in possession of the contents. 

`go-casper` implements H2O's CASPer and provides similar fucntinality in any golang http server. It wraps go's standard server push method (see ["HTTP/2 Server Push · Go, the unwritten parts"](https://rakyll.org/http2push/) if you don't how to use it) and maintains a fingerprint of browser caches and decides to push or cancel. The fingerprint is generated by using [golomb-coded sets](internal/encoding/golomb) (a compressed encoding of Bloom filter). 

The full documentation is available on [Godoc][godocs].

*NOTE1*: The project is still under heavy implementation. API may be changed in future and documentaion is incomplete. 

*NOTE2*: There is a [draft](https://datatracker.ietf.org/doc/draft-kazuho-h2-cache-digest/) by H2O author which defines a HTTP/2 frame type to allow clients to inform the server of their cache's contents. This can be replace with this in future. 

## Example 

Below is a simple example of usage.

```golang
// Initialize casper with false-positive probability 
// 1/64 and number of assets 10.
pusher := casper.New(1<<6, 10)

http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {    
    
    // Execute cache aware server push. 
    // 
    // In this example, it generates a fingerprint "JA" and set it
    // as "x-go-casper" cookie value.
    // 
    // If you access this handler first time, it runs server-push.
    // But from next time, with same client, it cancels pushing since 
    // cookie indicates asset has already been cached by the client.
    if _, err := pusher.Push(w, r, []string{"/static/example.js"}, nil); err != nil {
        log.Printf("[ERROR] Failed to push assets: %s", err)
    }

    // ...
})
```

You can find the complete example [here](/example).