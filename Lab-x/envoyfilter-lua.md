## EnvoyFilter-Lua使用实验


1. 创建新的namespace 

```
# kubectl create ns istioinaction
# kubectl label namespace istioinaction istio-injection=enabled
```

2. 部署待测试应用

```
# kubectl -n istioinaction apply -f httpbin.yaml
# kubectl -n istioinaction apply -f bucket-tester-service.yaml
# kubectl -n istioinaction apply -f sleep.yaml
```
3. 应用EnvoyFilter

```
# kubectl apply -f lua-filter.yaml
```

4. 测试验证

```
# kubectl -n istioinaction  exec -it deploy/sleep  -- curl -vv httpbin.istioinaction:8000/headers
*   Trying 10.96.96.25:8000...
* Connected to httpbin.istioinaction (10.96.96.25) port 8000 (#0)
> GET /headers HTTP/1.1
> Host: httpbin.istioinaction:8000
> User-Agent: curl/7.83.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< server: envoy
< date: Wed, 01 Mar 2023 08:20:53 GMT
< content-type: application/json
< content-length: 588
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 4
< istioinaction: it works!
<
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.istioinaction:8000",
    "User-Agent": "curl/7.83.1",
    "X-B3-Parentspanid": "74b6dccbd3aff2b4",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "e71af86df58e586a",
    "X-B3-Traceid": "810b6744dbb6c4a974b6dccbd3aff2b4",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/istioinaction/sa/httpbin;Hash=6401b8340a6343734e2788a0e0afbfe3991fc9395a682d58bfedcb9e2fad9c48;Subject=\"\";URI=spiffe://cluster.local/ns/istioinaction/sa/sleep",
    "X-Test-Cohort": "dark-launch-7"
  }
}
* Connection #0 to host httpbin.istioinaction left intact

```

从终端可以看到，请求响应头中已经添加了`istioinaction: it works!`
```
            function envoy_on_response(response_handle)
              response_handle:headers():add("istioinaction", "it works!")
            end
```
另外，由于应用程序逻辑实现中，会将请求头中的信息返回，所以在响应body中可以看到`"X-Test-Cohort": "dark-launch-7"`

```
            function envoy_on_request(request_handle)
              local headers, test_bucket = request_handle:httpCall(
              "bucket_tester",
              {
                [":method"] = "GET",
                [":path"] = "/",
                [":scheme"] = "http",
                [":authority"] = "bucket-tester.istioinaction.svc.cluster.local",
                ["accept"] = "*/*"
              }, "", 5000)

              request_handle:headers():add("x-test-cohort", test_bucket)
            end
```
