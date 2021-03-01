# 認証
Synerexシステム用の認証機能の実装

## アーキテクチャー

Synerexシステム用の認証は下記のアーキテクチャーのように実装される。

```
                                                       +-----------------------+                 
                                                       |                       |                 
                      ① Authorization grant            |                       |                 
            +------------------------------------------>      Synerex oAuth    |                 
            |                                          |                       |                 
            |                                          |                       |                 
            |                                          +-----------------------+                 
            |                                                      |                             
+-----------|-----------+                                          |                             
|                       |            ② Access token                |                             
|                       <------------------------------------------+                             
|        Client         |                                                                        
|                       |     ③ Send request with access token                                   
|                       |------------------------------------------+                             
+-----------------------+                                          |                             
            ^                                                      |                             
            |                                          +-----------v-----------+                 
            |                                          |                       |                 
            |          ④ Protected resource            |                       |                 
            +------------------------------------------|     SynerexServer     |                 
                                                       |                       |                 
                                                       |                       |                 
                                                       +-----------------------+                 
                                                                                                 
                                                                                                 
                                                                                                 
```

## コンフィギュア

### 環境変数
Synerexサーバーを実行する前に下記の環境変数を設定する。

```shell script
# set your public key
export SX_PUBLIC_KEY=8AAC2286D4448FAD779F147357BFF
# set you private key
export SX_PRIVATE_KEY=61268C659CAADC95C8DFB41399C12
```

### Dockerを使う
Dockerを使う場合は`docker-compose.yml`の中に `SX_PUBLIC_KEY` `SX_PRIVATE_KEY`の変数を設定する。
```yaml
    environment:
      - SX_SERVER_HOST=server
      - SX_NODESERV_HOST=nodeserv
      # set your public key
      - SX_PUBLIC_KEY=8AAC2286D4448FAD779F147357BFF
      # set you private key
      - SX_PRIVATE_KEY=61268C659CAADC95C8DFB41399C12
```

## Sample

Get access token
```go
cc, err := grpc.Dial(*serverAddress, transportOption)
if err != nil {
    log.Fatal("cannot dial server: ", err)
}

defer cc.Close()
auth := makeAuthRequest()
client := api.NewSynerexClient(cc)
authResponse, err := client.GetAccessToken(context.Background(), auth)
if err != nil {
    log.Fatal("get access token error: ", err)
}
log.Println("access token", authResponse.AccessToken)
```


Access to protected resource
```go
cc, err := grpc.Dial(*serverAddress, transportOption)
if err != nil {
    log.Fatal("cannot dial server: ", err)
}

defer cc.Close()
req := makeRequest()
md := metadata.New(map[string]string{
    "authorization": "Bearer accessToken",
})
ctx := metadata.NewOutgoingContext(context.Background(), md)
client := api.NewSynerexClient(cc)
response, err := client.CallToSomeResource(ctx, req)
```

## Unit Test
下記はUnitTestの結果になる。

```
2021/03/01 17:17:32 Register Metrics
=== RUN   TestValidToken
2021/03/01 17:17:32 hmac token is required
2021/03/01 17:17:32 token not match. token: 1234567890, expected token: ce977846b410395a1d8c979097e9c43b
--- PASS: TestValidToken (0.00s)
=== RUN   TestGetRsaPrivateKey
2021/03/01 17:17:32 read file error open ./test/private.key: no such file or directory
2021/03/01 17:17:32 parse rsa key fail Invalid Key: Key must be PEM encoded PKCS1 or PKCS8 private key
--- PASS: TestGetRsaPrivateKey (0.00s)
=== RUN   TestGenerateAccessToken
--- PASS: TestGenerateAccessToken (0.00s)
=== RUN   TestVerify
2021/03/01 17:17:32 read file error open : no such file or directory
2021/03/01 17:17:32 get verify key fail Invalid Key: Key must be PEM encoded PKCS1 or PKCS8 private key
2021/03/01 17:17:32 parse token fail token contains an invalid number of segments
2021/03/01 17:17:32 parse token fail Token is expired
2021/03/01 17:17:32 invalid access token, role invalid 
--- PASS: TestVerify (0.01s)
=== RUN   TestAccessibleRoles
--- PASS: TestAccessibleRoles (0.00s)
=== RUN   TestAuthorize
2021/03/01 17:17:32 metadata map[]
2021/03/01 17:17:32 parse token fail token contains an invalid number of segments
--- PASS: TestAuthorize (0.01s)
PASS
coverage: 10.8% of statements in ../../synerex/...
```

