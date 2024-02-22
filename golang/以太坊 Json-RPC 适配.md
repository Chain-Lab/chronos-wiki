# Json-RPC 适配与 Blockscout 部署

`blockscout version: 6.0.0`

为了直接使用现有的区块链浏览器 Blockscout，需要适配一系列的 JSON-RPC 接口

## 接口适配

为了最终实现一个形式上的 ethereum 的 JSON-RPC 接口，需要经过一系列的接口适配：

1. Karmem -> RPC(Protobuf)：Chronos 内部的数据结构通过 Karmem 实现，而 RPC 接口服务使用了 gRPC，gRPC 使用 Protobuf 来实现。第一步需要把区块、交易的数据转换到 Protobuf 中，这里暴露下面的一系列拉取区块、交易的接口，取得 Karmem 格式的数据后转换到 Protobuf

    ```protobuf
    syntax = "proto3";
    import "google/protobuf/empty.proto";
    option go_package="rpc/pb";
    
    service Blockchain {
      rpc GetBlockNumber(google.protobuf.Empty) returns (BlockNumberResp);
      rpc GetBlockByHash(GetBlockReq) returns (GetBlockResp);
      rpc GetBlockByNumber(GetBlockReq) returns (GetBlockResp);
      rpc GetTransactionByHash(GetTransactionReq) returns (GetTransactionResp);
      rpc GetTransactionByBlockHashAndIndex(GetTransactionReq) returns (GetTransactionResp);
      rpc GetTransactionByBlockNumberAndIndex(GetTransactionReq) returns (GetTransactionResp);
      rpc ReadContractAddress(ReadContractAddressReq) returns (ReadContractAddressResp);
      rpc SendTransactionWithData(SendTransactionWithDataReq) returns (SendTransactionWithDataResp);
    }
    ```

2. RPC(Protobuf) -> JSON-RPC：blockscout 需要调用一系列的 Ethereum 接口来拉取区块、交易数据，所必须的接口定义在 [Node Tracing / JSON RPC Requirements](https://docs.blockscout.com/for-developers/information-and-settings/requirements/node-tracing-json-rpc-requirements)

    Chronos 所实现的 RPC 接口本身就是为了实现服务的解耦，所以在暴露拉取区块、交易的 RPC 后，这里使用 python 的 JSON-RPC 来提供 openapi 实现类似 ethereum 的调用接口，具体实现见 [chronos-openapi](https://github.com/Chain-Lab/chronos-openapi)：

    ```python
    @blockchainApi.dispatcher.add_method
    def eth_getBlockByNumber(*args, **kwargs):
        # print(args)
        client = BlockchainRPC()
        if args[0] == "latest":
            resp = client.get_block_number()
            number = resp.number
        else:
            number = int(args[0], 16)
        full = args[1]
        resp = client.get_block_by_number(number, full)
    
        return block_adapter(resp, full)
    ```

## Blockscout 部署

部署流程：

* 克隆 blockscout 到本地：`git clone https://github.com/blockscout/blockscout.git`

* 安装 docker （如果有则直接跳过）: `sudo apt-get install docker.io`

* 安装最新版本的 docker-compose：

    ```bash
    VERSION=$(curl --silent https://api.github.com/repos/docker/compose/releases/latest | grep -Po '"tag_name": "\K.*\d')
    
    DESTINATION=/usr/local/bin/docker-compose
    sudo curl -L https://github.com/docker/compose/releases/download/${VERSION}/docker-compose-$(uname -s)-$(uname -m) -o $DESTINATION
    sudo chmod 755 $DESTINATION
    ```

* 启动容器：`sudo docker-compose up` (注意进入到 blockscout/docker-compose 目录)

**注意：**Blockscout 需要修改多个环境变量

1. 注释 `docker-compose.yml` 下 backend 中的环境变量 `ETHEREUM_JSONRPC_WS_URL`:

    ```
    environment:
        ETHEREUM_JSONRPC_HTTP_URL: http://host.docker.internal:8545/
        ETHEREUM_JSONRPC_TRACE_URL: http://host.docker.internal:8545/
          # ETHEREUM_JSONRPC_WS_URL: ws://host.docker.internal:8545/
        CHAIN_ID: '1337'
    ```

2. `env/common-frontend.env` 中，修改两个 host 为具体前端访问地址

    ```
    NEXT_PUBLIC_API_HOST = IP地址或域名
    NEXT_PUBLIC_APP_HOST = IP地址或域名
    ```

3. `env/common-blockscout.env` 中禁用一系列的索引

    ```
    DISABLE_INDEXER=false
    DISABLE_REALTIME_INDEXER=false
    DISABLE_CATCHUP_INDEXER=false
    INDEXER_DISABLE_TOKEN_INSTANCE_REALTIME_FETCHER=true
    INDEXER_DISABLE_TOKEN_INSTANCE_RETRY_FETCHER=true
    INDEXER_DISABLE_TOKEN_INSTANCE_SANITIZE_FETCHER=true
    INDEXER_DISABLE_TOKEN_INSTANCE_LEGACY_SANITIZE_FETCHER=true
    INDEXER_DISABLE_PENDING_TRANSACTIONS_FETCHER=true
    INDEXER_DISABLE_INTERNAL_TRANSACTIONS_FETCHER=true
    INDEXER_DISABLE_EMPTY_BLOCKS_SANITIZER=true
    INDEXER_DISABLE_ADDRESS_COIN_BALANCE_FETCHER=true
    INDEXER_DISABLE_CATALOGED_TOKEN_UPDATER_FETCHER=true
    INDEXER_DISABLE_BLOCK_REWARD_FETCHER=true
    ```