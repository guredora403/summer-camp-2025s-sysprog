# 02: ソケットプログラミング

## ソケットとは
今日の様々なアプリケーションでは、OSのネットワーク機能を活用して、インターネットを利用した多様な通信が行われています。このような通信のために、OSはソケット（socket）という仕組みを提供しています。

ソケットの考え方はシンプルです: こちらのプロセスでwriteしたデータを、あちらのプロセスでreadすることができるようになります。逆に、あちらのプロセスでwriteしたデータを、こちらのプロセスでreadできます。

お互いのプロセスは同じホスト（コンピュータ）上にあっても良いですし、違うホストにあっても構いません。そのような複雑な通信を、ソケットという仕組みでreadとwriteに抽象化してくれます。

ソケットでは、任意のバイナリデータをやりとりできます。つまり、どのような形式のデータを、どのような順番で送りあえばよいのかをお互いのアプリケーションが知っていれば、意味のある通信が可能になります。これが**プロトコル**です。逆に、プロトコルが一致しなければ、データの解釈が一致せずに、意味のある通信をすることはできません。

## さまざまなプロトコル
ここでは、OSI参照モデルにおけるアプリケーション層のプロトコルのみを扱います。トランスポート層までは、OSの抽象化を利用します。

アプリケーション層におけるプロトコルには、以下のようなものがあります。

- HTTP: hyper Text Transfer Protocol
- SMPT: Simple Mail Transfer Protocol
- DNS: Domain Name System
- FTP: File Transfer Protocol

## プロセスの識別

ソケットを使って通信をするときは、相手のプロセスを一いに識別して接続する必要があります。プロセスが実行されているコンピュータはIPアドレスで、プロセス自体はポートで識別します。

## UDPとTCP

ソケットが内部で利用するプロトコルには、**UDP**と**TCP**の二つがあります。厳密には、TCP事態もソケットで実現できますし、UDPもソケットで実現できるのですが、前述したようにOSがインターフェースを提供してくれているので、それを利用します。しかしながら、両社は扱い方がかなり異なるので、その違いを意識しておく必要があります。

UDP (User Datagram Protocol) は、コネクションレス型のプロトコルです。これは、データを送る前に相手と通信の確立（コネクション）を行わず、一方的にデータを送りつけるという特徴を持っています。手紙やハガキをポストに投函するのに似ています。送り手は宛先を書いて送るだけで、相手が受け取ったか、順番通りに読んだかまでは確認しません。

一方で、TCP (Transmission Control Protocol) は、コネクション指向型のプロトコルです。データを送る前に、相手との間で仮想的な通信路（コネクション）を確立し、お互いの状況を確認しながら通信を行います。確認用の通信が発生するため通信料は増えますが、確実に相手にデータが届くことが保証されます。

UDPは、光速であったり軽量であることが重視されるアプリケーションに利用されます。例として、動画ストリーミングや音声通話、DNSなどがあります。一方で、TCPは、メールの送受信や、ブラウザでのウェブサイトの取得に利用されています。

## 実践2-1: ncコマンドでhttp通信をしてみる

nc(netcat)は、TCP/UDPを用いて任意のデータをやりとりできるツールです。標準入力を相手に送り、相手から送られてきた内容が標準出力に表示されるようになっています。

これを使って、任意のHTTPサーバーに対してHTTPリクエストを送ってみましょう。HTTPは、ブラウザでウェブサイトを見るときに利用されているプロトコルです。

例えば、[阿部寛のホームページ](http://abehiroshi.la.coocan.jp/)を取得するには、以下のようにします。
```
nc -v abehiroshi.la.coocan.jp 80
```


接続が確立したら、以下のように入力します。
```
GET / HTTP/1.1
host:   abehiroshi.la.coocan.jp

```

このようなメッセージをやりとりして、ブラウザはウェブサイトを表示しています。

## ソケットで利用するシステムコール達

ソケットプログラミングでは、以下のようなシステムコールを利用します。
- socket: ソケットを作成する
- bind: ソケットを指定のポートに割り当てる
- listen: TCPの場合に、接続を待ち受ける
- accept: TCPの場合に、新しい接続を作成する
- connect: 指定のプロセスに接続する

## UDPソケットサーバの実装
UDP接続の場合、以下のようにサーバとして通信を待ち受けます:
- socketシステムコールでソケットを作成する。ファイルディスクリプタが帰る。
- bindシステムコールを使って、指定のポートにソケットを割り当てる
- recivefrom関数を使って、メッセージを受信する
- sendto関数を使って、メッセージを送信する
- closeで、ソケットを閉じる

### socket

socketは、ソケットを作成します。書式は以下の通りです:

```
#include <sys/socket.h>
#include  <netinet/ip.h>
int socket(int domain, int type, int protocol);
```

第１引数には、通信ドメインを指定します。今回はIPV4を利用するので、AF_INETを指定します。ほかに、AF_UNIXや、AF_LOCALが指定できますが、詳しい説明は省略します。

第２引数には、ソケットの種類を指定します。今回はUDPを利用するので、SOCK_DGRAMを指定します。TCPを利用する場合は、SOCK_STREAMになります。

第３引数には、利用するプロトコルを指定します。今回は、IPPROTO_UDPを指定します。

成功すると、新しいファイルディスクリプタが帰ります。-1が返ってきた場合は、エラーの発生を示します。

ソケットを作成しただけでは、まだどのポートにも割り当てられていないので、通信することはできません。ただ、口だけができた状態です。

### bind
bindは、ソケットを指定のポートに割り当てます。これを行うと、OSは指定のポートに届いた通信を、ソケットに転送するようになります。

```
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

第１引数には、ソケットのファイルディスクリプタを指定します。

第２引数には、ソケットを割り当てるsockaddr構造体へのポインタを指定します。sockaddr構造体については後述します。

第３引数にはsockaddr構造体のサイズを指定します。任意の変数や型をサイズを求めるには、sizeof演算子が利用できます。


### sockaddr構造体
sockaddr構造体は、ソケットで利用されるアドレス全般を表します。

```
#include <sys/socket.h>

struct sockaddr {
    sa_family_t     sa_family;      /* Address family */
    char            sa_data[];      /* Socket address */
};

```


このように、sockaddr構造体自体は、先頭にアドレスファミリが存在するだけで、その後に続くデータを規定していません。これは、共通のソケットAPIで、複数のアドレスファミリを扱えるようにするためです。

以下は、アドレスファミリと、それぞれのサイズを表にしたものです。

| アドレスファミリ | アドレス | ポート |
| ---------------- | -------- | ------ |
| IPV4             | 32bit    | 16bit  |
| IPV6             | 128bit   | 16bit  |

今回は、IPV4アドレスを利用しますが、もしIPV6アドレスを利用しなければならない場合でも、同じソケットAPIが利用できます。

以下は、IPV4/IPV6を表すsockaddr_inとsockaddr_in6構造体の定義です。

```
#include <netinet/in.h>

struct sockaddr_in {
    sa_family_t     sin_family;     /* AF_INET */
    in_port_t       sin_port;       /* Port number */
    struct in_addr  sin_addr;       /* IPv4 address */
};

struct sockaddr_in6 {
    sa_family_t     sin6_family;    /* AF_INET6 */
    in_port_t       sin6_port;      /* Port number */
    uint32_t        sin6_flowinfo;  /* IPv6 flow info */
    struct in6_addr sin6_addr;      /* IPv6 address */
    uint32_t        sin6_scope_id;  /* Set of interfaces for a scope */
};
```

両方とも、先頭のsa_family_t型のメンバが共通しています。このため、bindなどのシステムコールに渡すときは、sockaddr構造体のポインタ型にキャストして渡すだけで、カーネルが先頭のアドレスファミリを見てアドレスの内容を解釈してくれます。

### sockaddr_in構造体の作成
sockaddr_in構造体の各メンバに設定する値を説明します。

```
#include <netinet/in.h>

struct sockaddr_in {
    sa_family_t     sin_family;     /* AF_INET */
    in_port_t       sin_port;       /* Port number */
    struct in_addr  sin_addr;       /* IPv4 address */
};
```

sin_familyには、AF_INETを指定します。

sin_portには、ポート番号（１６ビット）を、ネットワークバイトオーダーで設定します。バイトオーダーについては後述します。今回は11111を指定します。

sin_addrには、IPV4アドレスを表すin_addr構造体が入っています。in_addr構造体のs_addrに、割り当てるIPアドレスをネットワークバイトオーダーで設定します。今回は、INADDR_ANYを指定します。

また、余計なデータの混入を防ぐため、構造体を宣言したら値を設定する前に、memset関数でクリアすることが一般的なようです。

```
#include <string.h>

void *
memset(void *b, int c, size_t len);
```

memsetは、指定の値で指定の領域を埋める関数です。第１引数に埋めたいメモリ領域の先頭のポインタ、第２引数に埋めたい値、第３引数にサイズを指定します。例えば、sockaddr_in構造体の変数sを0でクリアする場合、以下のようになります。

```
memset(&s, 0, sizeof(s));
```

### バイトオーダー
CPUの命令セットによって、複数バイトのデータをどの順番でメモリに読み書きするのかが異なっていて、これを**バイトオーダー**と呼びます。最も上位のバイトから先に書き込んでいくバイトオーダーはビッグエンディアン、下位のバイトから書き込んでいくバイトオーダーはリトルエンディアンと呼ばれています。

X86やARMなどの多くのCPUではリトルエンディアンですが、ネットワークでは、ビッグエンディアンのバイトオーダーを利用するように決められています。そのため、ポートやIPアドレスなど、必要に応じてネットワークバイトオーダーからホストのバイトオーダーへ、あるいはその逆へ、変換する必要があります。

そのために、htons/htonl/ntohs/ntohlのような関数が用意されています。
```
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);

uint16_t htons(uint16_t hostshort);

uint32_t ntohl(uint32_t netlong);

uint16_t ntohs(uint16_t netshort);
```

hはhostを、nはnetworkを表します。また、末尾のsはshort(16bit)、lはlong(32bit)を表します。

例えば、sin_portには、htons関数を使って変換したものを代入します。
```
s.sin_port = htons(11111);
```

### recvfrom
recvfromは、ソケットからメッセージを受け取ります。
```
#include <sys/types.h>
#include <sys/socket.h>


ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

sockfdには、ソケットのファイルディスクリプタを指定します。

bufには、ソケットからのデータを受信するバッファへのポインタを指定します。

lenには、バッファサイズを指定します。

flagsには動作フラグを指定します。今回は０を指定します。

src_addrには、送信元アドレスを受け取るためのsockaddr構造体へのポインタを指定します。事前に、sockaddr_in構造体を宣言し、sockaddr構造体へのポインタにキャストして渡すことが、送信元アドレスが書き込まれます。

addrlenには、src_addrのサイズが書かれたメモリアドレスへのポインタを指定します。

成功すると、読み込まれたバイト数が、失敗すると-1が帰ります。

### sendto
