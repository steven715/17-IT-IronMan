# 兵荒馬亂的第一天

今天是第17屆的鐵人賽，這次參加有點零時，算是給自己趕鴨子上架，所以目前是還無準備的狀況，

那如果以這一年來說，其實滿多專注在linux的系統上面，所以這邊就來介紹一下工作上常用的一些linux工具以及其指令。

## tail

- 輸出檔案最後幾行: `tail -n 10`
- 串流輸出: `tail -f`

## find

- 根據時間，天: `find ./ -mtime +1 -exec rm -fr {} \;`
- 根據時間，分鐘: `find ./ -mmin +15 -delete`
- 搭配正規表達式: `find ./ -regex “.*”`
- 根據名稱: `find / -name kafka-console-consumer.sh 2>/dev/null`

## 查看崩潰的服務

- `journalctl`
- `dmsg`
- `/var/log/messages`

## addr2line

- 可以從異常紀錄的callstack 中，找出具體錯誤行數，callstack: 0x7f80950b068a

`#addr2line -e 執行檔 0x7f80950b068a`

## readelf

- 看ELF 檔案的程序結構與符號鏈接內容
- `readelf -a 執行檔`

## objdump

- 用於查看二進制文件內容的工具

## nohup

- 背景執行: `nohup exe &`
- 不要輸出到標準輸出: `nohup exe > /dev/null 2>&1 &`

## sed

- Sample: `sed -i ‘s/old/new/g’ xxx.txt`
- -i: 直接修改檔案內容，不產生中間檔案
- ‘s/old/new/g’
    - s: 字串
    - g: 全域替換

## file

- `file *` ⇒ 查看目錄下所有檔案的編碼格式

## iconv

- `iconv -f GB2312 -t UTF-8 xxx.h -o xxx.h` ⇒ 轉換目標檔案的編碼格式

## ldd

- `ldd xxx.lib` ⇒ 查看該庫或執行文件的依賴庫

## split

- `split -d -b 500m large_log.log small_log_` ⇒ 將大檔案切分小檔案

## timeout

- `timeout 30m xxx.sh` ⇒ 執行 xxx.sh 腳本，30分鐘後關閉

## tcpkill

- `tcpkill -i ens33 host 10.10.100.113 and port 9092` ⇒ 模擬網路斷開現象

### timedatectl

- 用來做時間管理與同步
- `timedatectl set-timezone Asia/Taipei` : 換時區

## iptable

- `iptables -A INPUT -s 8.218.254.34 -p tcp --dport 8886 -j DROP` ⇒ 屏蔽指定的ip, port來的封包
- `iptables -D INPUT -s 8.218.254.34 -p tcp --dport 8886 -j DROP` ⇒ 恢復指定的ip, port來的封包

## tar

- `tar zxvf {File.tar.gz}` ⇒ 解壓縮
- `tar zcvf {File.tar.gz} {資料夾名稱}` ⇒ 壓縮

