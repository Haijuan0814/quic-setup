
Emulab Experiment
#  0. Tunning Make socker buffers larger Increase bytes limit at dummynet

# 调整 dummynet 的管道字节限制，管道的字节限制设置为 10,000,000 字节。dummynet 是 FreeBSD 系统中的一个流量管理工具，用于网络带宽控制和模拟延迟、丢包等网络条件。
sudo sysctl net.inet.ip.dummynet.pipe_byte_limit=10000000
# 增加 socket 缓冲区的最大值：将内核 IPC（进程间通信）中 socket 缓冲区的最大值设置为 16,777,216 字节（16MB）
sudo sysctl kern.ipc.maxsockbuf=16777216
# 设置 TCP 发送和接收缓冲区的大小： 12,582,912 字节（12MB）。较大的缓冲区可以处理更多的数据，从而减少传输延迟，提高吞吐量
sudo sysctl net.inet.tcp.sendspace=12582912
sudo sysctl net.inet.tcp.recvspace=12582912

# 设置 TCP 的内存使用参数：
   # tcp_wmem：用于发送缓冲区的内存大小
   # tcp_rmem：用于接收缓冲区的内存大小
   # tcp_mem：整个 TCP 堆栈的内存使用限制。
   # 三个值分别表示最小内存（10,000,000 字节），默认内存（30,000,000 字节）和最大内存（50,000,000 字节）
echo "10000000 30000000 50000000" > /proc/sys/net/ipv4/tcp_wmem 
echo "10000000 30000000 50000000" > /proc/sys/net/ipv4/tcp_rmem
echo "10000000 30000000 50000000" > /proc/sys/net/ipv4/tcp_mem

# 调整核心内存缓冲区的默认值和最大值
echo 30000000 > /proc/sys/net/core/wmem_default
echo 50000000 > /proc/sys/net/core/wmem_max
echo 30000000 > /proc/sys/net/core/rmem_default
echo 50000000 > /proc/sys/net/core/rmem_max

#  1. Instatiate 'quic-fec' Profile to Experiment Give project and path in parameters section

#  2. Login to nodes
   5 terminals, 2 server , 2 client. 1 link bridge

#  3.Check Apache Status
   nmuthura@server:/proj/FEC-HTTP/long-quic$ service apache2 status
#  4.Generate SPKI for TCP
   # Get Apache spki 
    nmuthura@server:/proj/FEC-HTTP/long-quic$ openssl x509 -pubkey < "/etc/ssl/certs/apache-selfsigned.crt" | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64 > "apache-selfsigned-spki.txt"

    # Get QUIC spki
    nmuthura@server:/proj/FEC-HTTP/long-quic/long-look-quic/quic/out$ openssl x509 -pubkey < "leaf_cert.pem" | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64 > "server_pub_spki.txt"
#  5. TCP Test
   Apache Server is always running as system service.


## Client

#  Terminal 1
nmuthura@client:/proj/FEC-HTTP$ /proj/FEC-HTTP/long-quic/chrome-linux/chrome111  --no-sandbox --headless --disable-gpu --remote-debugging-port=9222  --user-data-dir=/tmp/chrome-profile --enable-benchmarking --enable-net-benchmarking --ignore-certificate-errors-spki-list=$(cat /proj/FEC-HTTP/long-quic/apache-selfsigned-spki.txt)  --no-proxy-server   --disable-quic    --host-resolver-rules='MAP www.example-tcp.org 192.168.1.1'

# Terminal 2
nmuthura@client:~$ chrome-har-capturer --force --port 9222 -o tcp_index.har https://www.example-tcp.org/
- https://www.example-tcp.org/ ✓
nmuthura@client:~$ chrome-har-capturer --force --port 9222 -o tcp_5k.har https://www.example-tcp.org/5k.html
- https://www.example-tcp.org/5k.html ✓
nmuthura@client:~$ vim tcp_index.har 
# QUIC Test



## Server

nmuthura@server:/proj/FEC-HTTP/long-quic/quiche/bazel-bin/quiche$ ./quic_server --quic_response_cache_dir=/proj/FEC-HTTP/long-quic/long-look-quic/quic/www.example-quic.org   --certificate_file=/proj/FEC-HTTP/long-quic/long-look-quic/quic/out/leaf_cert.pem --key_file=/proj/FEC-HTTP/long-quic/long-look-quic/quic/out/leaf_cert.key

For Debugging --v=1 --stderrthreshold=0

## Client

# Terminal 1
nmuthura@client:/proj/FEC-HTTP$ /proj/FEC-HTTP/long-quic/chrome-linux/chrome111 --no-sandbox --headless --disable-gpu --remote-debugging-port=9222  --user-data-dir=/tmp/chrome-profile1  --enable-benchmarking --enable-net-benchmarking --ignore-certificate-errors-spki-list=$(cat /proj/FEC-HTTP/long-quic/long-look-quic/quic/out/server_pub_spki.txt)  --no-proxy-server   --enable-quic   --origin-to-force-quic-on=www.example-quic.org:443   --host-resolver-rules='MAP www.example-quic.org:443 192.168.1.1:6121'
For Debugging --enable-logging --v=3

# Terminal 2
nmuthura@client:~$ chrome-har-capturer --force --port 9222 -o quic_index.har https://www.example-quic.org/
- https://www.example-quic.org/ ✓
nmuthura@client:~$ chrome-har-capturer --force --port 9222 -o quic_5k.har https://www.example-quic.org/5k.html
- https://www.example-quic.org/5k.html ✓
nmuthura@client:~$ vim quic_5k.har 

# Verify response protocol , status and size in HAR file

# 7.Experiment
  ## (1) Start Iperf
nmuthura@server:/proj/FEC-HTTP/long-quic$ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------

  ## (2)Start quic server
nmuthura@server:/proj/FEC-HTTP/long-quic/quiche/bazel-bin/quiche$ ./quic_server --quic_response_cache_dir=/proj/FEC-HTTP/long-quic/long-look-quic/quic/www.example-quic.org   --certificate_file=/proj/FEC-HTTP/long-quic/long-look-quic/quic/out/leaf_cert.pem --key_file=/proj/FEC-HTTP/long-quic/long-look-quic/quic/out/leaf_cert.key
  ## (3)Activate python env
nmuthura@client:/proj/FEC-HTTP$ source nenv/bin/activate
(nenv) nmuthura@client:/proj/FEC-HTTP$ 
  ## (4)Update Script for defaults
vim

  ## (5)Start experiment
(nenv) nmuthura@client:/proj/FEC-HTTP/long-quic/long-look-quic/test_src$ python engineWrapper.py  --browserPath=/proj/FEC-HTTP/long-quic/chrome-linux/chrome111 --mainDir='/proj/FEC-HTTP/long-quic/long-look-quic/data/demo1' --quic-version=RFCv1 --rounds=1 --networkInt=enp7s0f0 --experiment=demo --rates="10_36_0,50_36_0" --indexes="5k,10k"

# 8. Extracting Results
The experiment will output HAR files for each runs, we have a script which loads all these HAR files , extracts the page load times , subtracts the DNS times and reports the Welch's statistical significance result . Finally computes the average over the total number of runs and produces time for TCP and QUIC.

Parameters are hardcoded in the script, change them before excuting

(nenv) nmuthura@client:/proj/FEC-HTTP/long-quic/long-look-quic/test_src$ python getTimes.py 
Setting : X_36_0
Object : 5k.html
1.3816038678505005 1.1946499908307662
Performance Difference Statistically Significant
HTTPS:  180.82444999995977
QUIC:  146.52499999960665

Object : 10k.html
1.1461368701626866 1.461937495430736
Performance Difference Statistically Significant
HTTPS:  184.13365000007002
QUIC:  150.07939999932947
....

QUIC Version :  ../data/tcpBBR_quicV1BBRv2/
Setting : X_36_0
Total Runs : 20
TCP Times
     5k.html   10k.html  100k.html  200k.html  500k.html    1mb.html   10mb.html
0  180.82445  184.13365   284.8286  375.68580  640.38365  1055.98375  8616.63475
1  174.20595  175.99035   272.0083  306.43845  365.19035   453.51265  1987.28875
2  174.51155  175.78580   270.7998  305.18360  355.01590   459.12400  1193.18840

['180.82445 184.13365 284.8286 375.68580 640.38365 1055.98375 8616.63475',
 '174.20595 175.99035 272.0083 306.43845 365.19035  453.51265 1987.28875',
 '174.51155 175.78580 270.7998 305.18360 355.01590  459.12400 1193.18840']

QUIC Times
    5k.html   10k.html  100k.html  200k.html  500k.html   1mb.html   10mb.html
0  146.5250  150.07940  228.99785  315.23850  578.76270  991.67750  8605.78995
1  141.9913  143.39385  209.35405  249.34725  311.20360  417.06865  1959.42185
2  142.5145  142.60395  209.19610  249.06035  306.50985  354.72555  1395.88385

['146.5250 150.07940 228.99785 315.23850 578.76270 991.67750 8605.78995',
 '141.9913 143.39385 209.35405 249.34725 311.20360 417.06865 1959.42185',
 '142.5145 142.60395 209.19610 249.06035 306.50985 354.72555 1395.88385']

# 9. Plotting Results
The TCP and QUIC times produced by the script are pasted into the heatmap ploting script to get the plots
