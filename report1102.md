#課題内容(ルータの CLI を作ろう)
>
>ルータのコマンドラインインタフェース (CLI) を作ろう。
>
>次の操作ができるコマンドを作ろう。
>
>* ルーティングテーブルの表示
>* ルーティングテーブルエントリの追加と削除
>* ルータのインタフェース一覧の表示
>* そのほか、あると便利な機能
>
>コントローラを操作するコマンドの作りかたは、第3回パッチパネルで作った patch_panel コマンドを参考にしてください。

#解答
##0. コマンドの仕様
今回の課題では，実行用のバイナリとして/bin/simple_routerを用意し，要求されたコマンドはこのバイナリに引数としてサブコマンドを与えて実行するものとした．また、コマンドの実行結果は，Trema runプロセスが実行されている端末ではなく，このバイナリが実行される端末に表示するものとした．

各サブコマンドの仕様は以下に定めた．

1. ルーティングテーブルの表示
  * [使用方法]`/bin/simple_router show_routing_table`
1. ルーティングテーブルエントリの追加
  * [使用方法]`/bin/simple_router add_entry 宛先ip ネットマスク 転送先`
  * 宛先ip、転送先は8ビットずつ10進表記
  * ネットマスクは数値で指定
1. ルーティングテーブルエントリの削除
  * [使用方法]`/bin/simple_router del_entry 宛先ip ネットマスク`
  * 宛先ipは8ビットずつ10進表記
  * ネットマスクは数値で指定
1. ルーターのインターフェース一覧の表示
  * [使用方法]`/bin/simple_router show_interface`

##1. ルーティングテーブルの表示
###1.1 コマンドの設計方針
取得したいルーティングテーブルの情報はRoutingTableクラス(/lib/routing_table.rbで実装)のインスタンス変数@dbで管理されているので、この情報を表示するようにコマンドを設計する．
RoutingTableクラスのインスタンスはTrema runプロセスによって起動されるSimpleRouterクラスのインスタンス(/lib/simple_router.rbで実装)がインスタンス変数@routing_tableとして管理しているので、以下の処理を各ファイルに実装した．

* `/bin/simple_router`   : SimpleRouter#show_RT()を呼び出すコマンドshow_routing_tableを実装
* `/lib/simple_router.rb`: RoutingTable#list()を呼び出すメソッドshow_RT()を実装
* `/lib/routing_table.rb`: @dbとネットマスクの最大値を返すメソッドlist()を実装

コマンド実行時の呼び出し関係は以下のようになる．
1. 実行用バイナリ/bin/simple_router(自作)がTrema runプロセスによって起動されるSimpleRouterクラスのインスタンスが持つshow_RTメソッドを呼び出す
2. SimpleRouterクラスのshow_RTメソッドがRoutingTableクラスのインスタンス変数@routing_tableのインスタンスメソッドlistを呼び出す
3.


⑤ルーティングテーブルの内容を表示  
↓①SimpleRouter#show_RT()  ↑④ルーティングテーブルの内容、最大マスク長  
Trema runプロセスによって起動されるSimpleRouter  
↓②RoutingTable#list()     ↑③ルーティングテーブルの内容、最大マスク長  
RoutingTable



###1.2 コマンドの実装
* `/bin/simple_router`(対応部分のみ抜粋)
```ruby
include Pio

desc 'List the Routing Table'
command :show_routing_table do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
  c.action do |_global_options, options, args|

    @db, @length = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      show_RT()

    print "destination \t\t  next hop\n"

    @length.downto(0).each do |eachMask|
      @db[eachMask].each do |dest, next_hop|
        print IPv4Address.new(dest).to_s+"/"+eachMask.to_s+"\t\t"+next_hop.to_s+"\n"
      end
    end

  end
end
```
* `/lib/simple_router.rb`(対応部分のみ抜粋)
```ruby
def show_RT()
  logger.info "show_routing_table() is called"
  return @routing_table.list()
end
```
* `/lib/routing_table.rb`(対応部分のみ抜粋)
```ruby
def list()
  return @db, MAX_NETMASK_LENGTH
end
```

##2. ルーティングテーブルエントリの追加と削除

##3. ルーターのインターフェース一覧の表示
