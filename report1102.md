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

コマンド実行時の呼び出し関係を以下の図に示す．
![図１](https://github.com/handai-trema/simple-router-d-miura/blob/master/fig1.png)

<!--
コマンド実行時の呼び出し関係は以下のようになる．
1. 実行用バイナリ/bin/simple_router(自作)がTrema runプロセスによって起動されるSimpleRouterクラスのインスタンスが持つshow_RTメソッドを呼び出す
2. SimpleRouterクラスのshow_RTメソッドがRoutingTableクラスのインスタンス変数@routing_tableのインスタンスメソッドlistを呼び出す
3.


⑤ルーティングテーブルの内容を表示  
↓①SimpleRouter#show_RT()  ↑④ルーティングテーブルの内容、最大マスク長  
Trema runプロセスによって起動されるSimpleRouter  
↓②RoutingTable#list()     ↑③ルーティングテーブルの内容、最大マスク長  
RoutingTable
-->


###1.2 コマンドの実装内容
* `/bin/simple_router`(対応部分のみ抜粋)  
IPv4Addressクラスを利用するため、Pioをincludeした．  
```ruby
include Pio
```
SimpleRouter#show_RT()で取得したルーティングテーブルの内容を表示する処理を記述した．
```ruby
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

* `/lib/simple_router.rb`(追加部分のみ抜粋)  
RoutingTableクラスのインスタンス変数@routing_tableのインスタンスメソッドlistの結果を返すメソッドshow_RTを追加した．
```ruby
def show_RT()
  logger.info "show_routing_table() is called"
  return @routing_table.list()
end
```
* `/lib/routing_table.rb`(追加部分のみ抜粋)  
ルーティングテーブルの内容であるインスタンス変数@dbとネットマスクの最大長を返すメソッドlistを追加した
```ruby
def list()
  return @db, MAX_NETMASK_LENGTH
end
```
###1.3 実行結果

##2. ルーティングテーブルエントリの追加と削除
###2.1 コマンドの設計方針
###2.2 コマンドの実装内容
* `/bin/simple_router`(対応部分のみ抜粋)
```ruby
desc 'Add Entry to the Routing Table'
arg_name 'destination_ip, netmask, forward_to'
command :add_entry do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask = args[1].to_i
    next_hop = args[2]
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      add_routing_table(destination_ip, netmask, next_hop)
  end
end

desc 'Delete Entry to the Routing Table'
arg_name 'destination_ip, netmask, forward_to'
command :del_entry do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

  c.action do |_global_options, options, args|
    destination_ip = args[0]
    netmask = args[1].to_i
    Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      del_routing_table(destination_ip, netmask)
  end
end
```
* `/lib/simple_router.rb`(追加部分のみ抜粋)
```ruby
def add_routing_table(destination_ip, netmask, next_hop)
  logger.info "add_routing_table() is called"
  @routing_table.add({:destination => destination_ip, :netmask_length => netmask, :next_hop => next_hop})
end

def del_routing_table(destination_ip, netmask)
  logger.info "del_routing_table() is called"
  @routing_table.del({:destination => destination_ip, :netmask_length => netmask})
end
```
* `/lib/routing_table.rb`(追加部分のみ抜粋)
```ruby
def del(options)
  netmask_length = options.fetch(:netmask_length)
  prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
  @db[netmask_length].delete(prefix.to_i)
end
```
###2.3 コマンドの実行結果

##3. ルーターのインターフェース一覧の表示
###3.1 コマンドの設計方針
###3.2 コマンドの実装内容
* `/bin/simple_router`(対応部分のみ抜粋)
```ruby
require './lib/interface'
```
```ruby
desc 'List the Interface of Router'
command :show_interface do |c|
  c.desc 'Location to find socket files'
  c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
  c.action do |_global_options, options, args|
    interfaces = Trema.trema_process('SimpleRouter', options[:socket_dir])
                      .controller
                      .show_IF()
    print "port_number\tmac_address\t\tip_address/netmask\n"
    interfaces.each do |each|
      print each.port_number.to_s+"\t\t"+each.mac_address.to_s+"\t"+each.ip_address.to_s+"/"+each.netmask_length.to_s+"\n"
    end
  end
end
```
* `/lib/simple_router.rb`(追加部分のみ抜粋)
```ruby
def show_IF()
  logger.info "show_interface() is called"
  return Interface.all
end
```
###3.3 コマンドの実行結果
