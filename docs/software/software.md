# Software

Here, the author describes the hul-common-lib and the rayraw-soft.

## hul-common-lib

[Github repository](https://github.com/spadi-alliance/hul-common-lib)

Since the basic structure of The RAYRAW firmware is the same that of AMANEQ's firmware, the RAYRAW board can be controlled by hul-common-lib.
Before reading this section, see the [AMANEQ user guide](https://spadi-alliance.github.io/ug-amaneq/software/software/).
In this user guide, the author additionally explains about programs for RAYRAW.

## rayraw-soft

[Github repository](https://github.com/spadi-alliance/rayraw-soft)

`rayraw-soft`はRAYRAWのファームウェア専用の機能を提供します。
コンパイルするためには`hul-common-lib`が必要です。

### Executable files

#### TrgFW/daq

Triggered-type ADC multi-hit TDC専用のデータ収集用プログラムです。
引数で指定したサーチ窓をADCとTDCに設定し、指定したイベント数データを取得します。
指定したイベント数取得した後自動的にプログラムは終了しますが、`Ctr-C`で割り込んで止める事も可能です。
プログラムを実行した場所に`data`ディレクトリがある必要があります、あらかじめ作成しておいてください。

このプログラムは1台のボードからデータを取得するための簡易的なDAQプログラムです。
**このプログラムを使って物理実験を行う事は想定してません。必ずお使いのDAQソフトウェアへ機能を移植してください。**
このプログラム内では

- トリガー種類の選択
- トリガー入力をNIM-IN1へアサイン
- TDCとADCへのサーチ窓の設定
- イベントカウンタのリセット
- ADCブロックの初期化状況の確認

を行ってからTCP接続を行いデータ読み出しを開始します。
上記はRAYRAWボードの制御であり、データ読み出しのシーケンスとは独立のため1つのプログラム内で全て行う必要は本来ありません。
このプログラムでは簡単のために全てを1つの実行体内で行っていますが、移植の際には制御部は適宜分離してください。

```shell
    [IP]           [Run No] [Num of events] [Window max] [Window min]
daq 192.168.10.16  1        10000           500              300
```

上記の例ではサーチ窓を300から500にセットし、10000イベントを`./data/run1.dat`というファイルへ保存します。
サーチ窓のLSB精度およびビット幅はファームウェアのシステムクロック周期に依存します。
ファームウェアの項目を参照してください。
ここで指定した値がTDCとADC両方に対して使用されます。
サーチ窓値のLSB精度をA nsとすると、上記例ではコモンストップ信号からみて`300*A - 500*A`秒過去のデータが取得できます。


#### YaenamiControl/set\_asic\_register

YAENAMI ASICにレジスタ値を送信するためのプログラムです。レジスタ値は後述の`rayraw-control`で作成します。
このプログラムには、InitializeとNormalの2つのモードがあります。


- Initialize<br>
ADCのキャリブレーションを行うとともに、ユーザーが設定したい任意のレジスタを送信するモードです。
YAENAMI ASICは、電源投入後にADCのキャリブレーションを行う必要があります。**RAYRAWへの電源投入後に、必ず実行してください。**
ADCキャリブレーションの詳細については、[YAENAMI ASICのページ](../asic/yaenami.md)を参照してください。

- Normal<br>
ユーザーが設定したい任意のレジスタ値をYAENAMI ASICに送信するモードです。
`Initialize`モードでの実行後は、電源を切らない限り`Normal`モードのみの使用で問題ありません。


##### How to use
便利のために、`rayraw-soft/install/YaenamiControl/`内に`rayraw-control/RegisterValue`へのシンボリックリンクを作成しておくことを推奨します。
以下の実行例は、シンボリックリンクが`RegisterValue`という名前で作成されている前提です。

```shell
$ cd /path/to/rayraw-soft/install/YaenamiControl/
$ ls -l
drwxr-xr-x 2 hoge fuga 4.0K Apr  1 00:00 bin
drwxr-xr-x 2 hoge fuga 4.0K Apr  1 00:00 include
drwxr-xr-x 2 hoge fuga   59 Apr  1 00:00 RegisterValue -> /path/to/rayraw-control/RegisterValue/

                           [IP]           [Register file path]  [Mode]
$ ./bin/set_asic_register  192.168.10.16  RegisterValue         initialize
```

この例では、192.168.10.16(SiTCPのデフォルトIP)に対して、ADCキャリブレーションを含むレジスタの送信を行っています。
引数が足りない場合はUsageが出るようになっているので、引数を忘れてしまったときなどは適宜利用してください。



## rayraw-control

[GitHub Repository](https://github.com/spadi-alliance/rayraw-control.git)

`rayraw-control`は、RAYRAWボードに搭載されているYAENAMI ASICに設定するレジスタ値を作成するためのプログラムです。
実際にレジスタ値を反映させるには、`rayraw-soft`が必要になります。
なお、2025年4月1日現在、`rayraw-control`はYAENAMI ASIC v0のレジスタにのみ対応しています。

### How to use
#### インストール
上記のGitHub Repositoryからgit cloneをしてください。その後、makeするだけで使用できます。

```
$ git clone https://github.com/spadi-alliance/rayraw-control.git rayraw-control
$ cd rayraw-control
$ make
```

#### レジスタ値の作成


レジスタの設定ファイルは、`rayraw-control/setup`内にあります。
`setup_xxx.txt` (xxxはレジスタ名)の内容を書き換えることで、任意のレジスタの値を変更することができます。<br>
`setup_xxx.txt`の中には、そのレジスタに設定される値が32ch (RAYRAW 1台)分書かれています。
チャンネルは0 originになっており、

- 0 - 7 ch: YAENAMI ASIC-0
- 8 - 15 ch: YAENAMI ASIC-1
- 16 - 23 ch: YAENAMI ASIC-2
- 24 - 31 ch: YAENAMI ASIC-3

に対応しています。<br>
レジスタによっては8ch (ASIC 1枚)で共通のものがありますが、そのようなレジスタの場合はASICごとに値が書かれています。

ここでは、例としてYAENAMI ASICのVGAの入力段に接続されるコンデンサの数を設定する`setup_CIN_B_VGA.txt`を挙げます。

```
# setup of CIN_B_VGA
# Range: 1 - 4 (2pF step)

# ch : Number of C connected (Default value: 4)

0 : 4
1 : 4
2 : 4
3 : 4
4 : 4
5 : 4
6 : 4
7 : 4
8 : 4
9 : 4
10 : 4
11 : 4
12 : 4
13 : 4
14 : 4
15 : 4
16 : 4
17 : 4
18 : 4
19 : 4
20 : 4
21 : 4
22 : 4
23 : 4
24 : 4
25 : 4
26 : 4
27 : 4
28 : 4
29 : 4
30 : 4
31 : 4
```

**`setup_xxx.txt`の中身は自由に書き換え可能ですが、ファイル名は変更しないください。エラーになります。**<br>
`rayraw-control`のV1.0は、RAYRAWの性能評価を行う際に作成されたプログラムであり、複数台のボードに対するレジスタ値を一度に作成する機能は現段階では有していません。
複数の設定ファイルが必要になる場合は、以下のような使い方を推奨します。

- `setup_xxx_board1.txt`, `setup_xxx_board2.txt`の様にボードごとに設定ファイルを作成し、`setup_xxx_txt`として該当のファイルにシンボリックリンクを貼る
- `setup_board1/`, `setup_board2/`の様にボードごとにディレクトリを作成し、`setup/`として該当のディレクトリにシンボリックリンクを貼る

書き換えが完了したら、その設定値を反映させたレジスタ値を作成します。

`rayraw-control/bin`の中にある`rayraw-control`を実行してください。引数は必要ありません。
`rayraw-control/RegisterValue/`の中に以下のような12個[^1]のテキストファイルが生成されていればOKです。
正常に生成されたことを確認するために、`ls -l`等でファイルの生成時間をチェックすることを推奨します。
```
$ cd rayraw-control
$ ./bin/rayraw-control
$ ls RegisterValue/
YAENAMI_1-1.txt  YAENAMI_1-3.txt  YAENAMI_2-2.txt  YAENAMI_3-1.txt  YAENAMI_3-3.txt  YAENAMI_4-2.txt
YAENAMI_1-2.txt  YAENAMI_2-1.txt  YAENAMI_2-3.txt  YAENAMI_3-2.txt  YAENAMI_4-1.txt  YAENAMI_4-3.txt
```

[^1]: YAENAMI ASICは電源投入後にADCのキャリブレーションが必要であり、その際にのみ使用するレジスタが存在します。そのため、1台あたり3つ、計12個のレジスタファイルが生成されるようになっています。

生成したレジスタファイルをYAENAMIに送るには、`rayraw-soft`に用意されている`YaenamiControl/set_asic_register`を使用します。
使い方については、[rayraw-soft](#rayraw-soft)をご覧ください。
