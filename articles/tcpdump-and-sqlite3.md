---
title: "tcpdump のキャプチャーを SQLite に取り込んでみる"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["network", "sqlite", "wireshark"]
published: true
---

# これは何か

クラウド全盛の時代でもやっぱり大活躍な tcpdump。ネットワークに関するトラブルには必須のツールです。
パケットキャプチャーは取得してからが本番なわけですが、シェルスクリプトやその他プログラミング言語を使ってごにょごにょし始めると、整形やフィルターに辛みを感じてきます。

そこで、プログラム的操作を減らすために、SQLite に取り込んでみます。

# 1. pcap から CSV への変換

SQLite では、CSV をそのままインポートできる素晴らしい機能があります。 ですので、一度 pcap ファイルを CSV にします。
CSV への変換はどんな方法でもいいのですが、Wireshark を使うのが一番簡単でしょう。

Wireshark では、画面に表示しているままのカラムで CSV にエクスポートできるので、必要に応じてカラムを追加しCSV にエクスポートします。

追加したいフィールドを右クリックして "列として適用" すると

![](https://storage.googleapis.com/zenn-user-upload/66beb1838a5a5dddfc67280b.png)

追加されました。

![](https://storage.googleapis.com/zenn-user-upload/c2fa6f9cff012e61f7d49fb7.png)

これをエクスポートします。

![](https://storage.googleapis.com/zenn-user-upload/c886377948709b38c1972b3b.png)
![](https://storage.googleapis.com/zenn-user-upload/eeed9d55be55a15c8a13753c.png)

無事エクスポートできました。

```
$ head zenn.csv
"No.","Time","Time Delta","Source","Destination","Protocol","Length","Info","Sequence Number","Next Sequence Number","Acknowledgment Number","Coloring Rule Name"
"1","2021-08-19 23:34:00.758915","2021-08-19 23:34:00.758915","172.18.225.3","172.18.224.1","TCP","71","38775  >  62715 [PSH, ACK] Seq=591882594 Ack=2357872891 Win=24571 Len=15","591882594","591882609","2357872891","TCP"
"2","2021-08-19 23:34:00.799190","2021-08-19 23:34:00.799190","172.18.224.1","172.18.225.3","TCP","56","62715  >  38775 [ACK] Seq=2357872891 Ack=591882609 Win=8212 Len=0","2357872891","2357872891","591882609","TCP"
"3","2021-08-19 23:34:01.420316","2021-08-19 23:34:01.420316","172.18.225.3","172.18.224.1","TCP","71","38775  >  54992 [PSH, ACK] Seq=166082811 Ack=269735427 Win=24571 Len=15","166082811","166082826","269735427","TCP"
"4","2021-08-19 23:34:01.461169","2021-08-19 23:34:01.461169","172.18.224.1","172.18.225.3","TCP","56","54992  >  38775 [ACK] Seq=269735427 Ack=166082826 Win=8211 Len=0","269735427","269735427","166082826","TCP"
"5","2021-08-19 23:34:01.929891","2021-08-19 23:34:01.929891","172.18.224.1","239.255.255.250","SSDP","145","M-SEARCH * HTTP/1.1 ","","","","UDP"
"6","2021-08-19 23:34:02.294112","2021-08-19 23:34:02.294112","172.18.224.1","172.18.225.3","TCP","75","62715  >  38775 [PSH, ACK] Seq=2357872891 Ack=591882609 Win=8212 Len=19","2357872891","2357872910","591882609","TCP"
"7","2021-08-19 23:34:02.294163","2021-08-19 23:34:02.294163","172.18.225.3","172.18.224.1","TCP","56","38775  >  62715 [ACK] Seq=591882609 Ack=2357872910 Win=24571 Len=0","591882609","591882609","2357872910","TCP"
"8","2021-08-19 23:34:02.421142","2021-08-19 23:34:02.421142","172.18.224.1","172.18.225.3","DNS","163","Standard query response 0x92d2 No such name PTR 160.122.171.52.in-addr.arpa SOA ns1-201.azure-dns.com","","","","UDP"
"9","2021-08-19 23:34:02.421142","2021-08-19 23:34:02.421142","172.18.224.1","172.18.225.3","DNS","163","Standard query response 0x92d2 No such name PTR 160.122.171.52.in-addr.arpa SOA ns1-201.azure-dns.com","","","","UDP"
```

:::message
Frame の Coloring Rule Name カラムを追加しておくと後でフィルターするときに便利です。
:::

# 2. SQLite に CSV をインポート

SQLite のインポートはワンライナーでできます。

```bash
# sqlite3 -separator <セパレーター> <データベースファイル> ".import <CSVファイルパス> <テーブル名>"
sqlite3 -separator , zenn.sqlite3 ".import zenn.csv zenn_cap"
```

ちゃんとインポートできたか見てみましょう。

```bash
$ sqlite3 zenn.sqlite3 ".schema zenn_cap"
CREATE TABLE zenn_cap(
  "No." TEXT,
  "Time" TEXT,
  "Time Delta" TEXT,
  "Source" TEXT,
  "Destination" TEXT,
  "Protocol" TEXT,
  "Length" TEXT,
  "Info" TEXT,
  "Sequence Number" TEXT,
  "Next Sequence Number" TEXT,
  "Acknowledgment Number" TEXT,
  "Coloring Rule Name" TEXT
);
```

素晴らしいことに、SQLite は インポートするときに CSV のヘッダーを見て、テーブルのスキーマを自動的に作ってくれます。

データの中身も見てみましょう。

## とりあえず 10 件
```bash
$ sqlite3 zenn.sqlite3 "select * from zenn_cap limit 10;"
1|2021-08-19 23:34:00.758915|2021-08-19 23:34:00.758915|172.18.225.3|172.18.224.1|TCP|71|38775  >  62715 [PSH, ACK] Seq=591882594 Ack=2357872891 Win=24571 Len=15|591882594|591882609|2357872891|TCP
2|2021-08-19 23:34:00.799190|2021-08-19 23:34:00.799190|172.18.224.1|172.18.225.3|TCP|56|62715  >  38775 [ACK] Seq=2357872891 Ack=591882609 Win=8212 Len=0|2357872891|2357872891|591882609|TCP
3|2021-08-19 23:34:01.420316|2021-08-19 23:34:01.420316|172.18.225.3|172.18.224.1|TCP|71|38775  >  54992 [PSH, ACK] Seq=166082811 Ack=269735427 Win=24571 Len=15|166082811|166082826|269735427|TCP
4|2021-08-19 23:34:01.461169|2021-08-19 23:34:01.461169|172.18.224.1|172.18.225.3|TCP|56|54992  >  38775 [ACK] Seq=269735427 Ack=166082826 Win=8211 Len=0|269735427|269735427|166082826|TCP
5|2021-08-19 23:34:01.929891|2021-08-19 23:34:01.929891|172.18.224.1|239.255.255.250|SSDP|145|M-SEARCH * HTTP/1.1 ||||UDP
6|2021-08-19 23:34:02.294112|2021-08-19 23:34:02.294112|172.18.224.1|172.18.225.3|TCP|75|62715  >  38775 [PSH, ACK] Seq=2357872891 Ack=591882609 Win=8212 Len=19|2357872891|2357872910|591882609|TCP
7|2021-08-19 23:34:02.294163|2021-08-19 23:34:02.294163|172.18.225.3|172.18.224.1|TCP|56|38775  >  62715 [ACK] Seq=591882609 Ack=2357872910 Win=24571 Len=0|591882609|591882609|2357872910|TCP
8|2021-08-19 23:34:02.421142|2021-08-19 23:34:02.421142|172.18.224.1|172.18.225.3|DNS|163|Standard query response 0x92d2 No such name PTR 160.122.171.52.in-addr.arpa SOA ns1-201.azure-dns.com||||UDP
9|2021-08-19 23:34:02.421142|2021-08-19 23:34:02.421142|172.18.224.1|172.18.225.3|DNS|163|Standard query response 0x92d2 No such name PTR 160.122.171.52.in-addr.arpa SOA ns1-201.azure-dns.com||||UDP
10|2021-08-19 23:34:02.421173|2021-08-19 23:34:02.421173|172.18.225.3|172.18.224.1|ICMP|191|Destination unreachable (Port unreachable)||||ICMP errors
```

## HTTP だけ
```bash
$ sqlite3 zenn.sqlite3 'select * from zenn_cap where "Coloring Rule Name" like "HTTP" limit 10;'
25|2021-08-19 23:34:05.482652|2021-08-19 23:34:05.482652|172.18.225.3|142.250.206.196|TCP|76|55370  >  80 [SYN] Seq=1885173153 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=1880965479 TSecr=0 WS=128|1885173153|1885173154|0|HTTP
26|2021-08-19 23:34:05.496954|2021-08-19 23:34:05.496954|142.250.206.196|172.18.225.3|TCP|76|80  >  55370 [SYN, ACK] Seq=10651938 Ack=1885173154 Win=65535 Len=0 MSS=1414 SACK_PERM=1 TSval=1122153421 TSecr=1880965479 WS=256|10651938|10651939|1885173154|HTTP
27|2021-08-19 23:34:05.497014|2021-08-19 23:34:05.497014|172.18.225.3|142.250.206.196|TCP|68|55370  >  80 [ACK] Seq=1885173154 Ack=10651939 Win=64256 Len=0 TSval=1880965493 TSecr=1122153421|1885173154|1885173154|10651939|HTTP
28|2021-08-19 23:34:05.497062|2021-08-19 23:34:05.497062|172.18.225.3|142.250.206.196|HTTP|146|GET / HTTP/1.1 |1885173154|1885173232|10651939|HTTP
29|2021-08-19 23:34:05.510158|2021-08-19 23:34:05.510158|142.250.206.196|172.18.225.3|TCP|68|80  >  55370 [ACK] Seq=10651939 Ack=1885173232 Win=65536 Len=0 TSval=1122153434 TSecr=1880965493|10651939|10651939|1885173232|HTTP
31|2021-08-19 23:34:05.595697|2021-08-19 23:34:05.595697|142.250.206.196|172.18.225.3|TCP|1470|80  >  55370 [ACK] Seq=10651939 Ack=1885173232 Win=65536 Len=1402 TSval=1122153520 TSecr=1880965493 [TCP segment of a reassembled PDU]|10651939|10653341|1885173232|HTTP
32|2021-08-19 23:34:05.595712|2021-08-19 23:34:05.595712|172.18.225.3|142.250.206.196|TCP|68|55370  >  80 [ACK] Seq=1885173232 Ack=10653341 Win=64128 Len=0 TSval=1880965592 TSecr=1122153520|1885173232|1885173232|10653341|HTTP
33|2021-08-19 23:34:05.595739|2021-08-19 23:34:05.595739|142.250.206.196|172.18.225.3|TCP|2872|80  >  55370 [ACK] Seq=10653341 Ack=1885173232 Win=65536 Len=2804 TSval=1122153520 TSecr=1880965493 [TCP segment of a reassembled PDU]|10653341|10656145|1885173232|HTTP
34|2021-08-19 23:34:05.595746|2021-08-19 23:34:05.595746|172.18.225.3|142.250.206.196|TCP|68|55370  >  80 [ACK] Seq=1885173232 Ack=10656145 Win=62720 Len=0 TSval=1880965592 TSecr=1122153520|1885173232|1885173232|10656145|HTTP
35|2021-08-19 23:34:05.596038|2021-08-19 23:34:05.596038|142.250.206.196|172.18.225.3|TCP|9882|80  >  55370 [ACK] Seq=10656145 Ack=1885173232 Win=65536 Len=9814 TSval=1122153520 TSecr=1880965493 [TCP segment of a reassembled PDU]|10656145|10665959|1885173232|HTTP
```

## サブクエリを使ってみる
実際調査をしたときのクエリをそのまま引っ張ってきたので何をやりたいのかよく分からないと思いますが。

```
sqlite3 tcpdump.sqlite3 "select * from tcpdump where [Acknowledgment Number] in (select [Next Sequence Number] from tcpdump where Info like '%10000%') and Info like '%FIN%';"
```

# 最後に

プログラムに全部任せるのもいいですが、整形したりフィルターしたりを SQL 文に閉じ込めることが出来るので、プログラムごとにその辺の処理を書かなくてよくなりました。