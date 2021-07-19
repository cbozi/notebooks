OCServ 配置 | Blog @ Jason

标题写的是 CentOS 7，其实 RedHat 系各发行版通用。  
ocserv 在 CentOS 6 上必须自行编译，且需要解决诸多依赖性问题，但在 CentOS 7 上配置相当容易。

#### 申请服务器证书

生成 CSR

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3 | cd ~  <br>  <br>openssl req -new -newkey rsa:4096 -sha256 -nodes -out server.csr -keyout server.key |

接下去的提示中，只有 Common Name 需要填写服务器域名，其他都可以留空。

不建议生成 ECC 证书，因为即使是正规 CA 签发，AnyConnect 客户端也会提示不安全。

拿着生成的 CSR 文件，到 Let’s encrypt 签发。  
如果签名算法可选，务必选择 SHA-2，不要用 SHA-1。

#### 安装 OCSERV

bash

|     |     |
| --- | --- |
| 1   | yum install epel-release ocserv -y |

#### 配置 OCSERV

bash

|     |     |
| --- | --- |
| 1   | vim /etc/ocserv/ocserv.conf |

修改如下：

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7  <br>8  <br>9  <br>10  <br>11  <br>12  <br>13  <br>14  <br>15  <br>16  <br>17  <br>18  <br>19  <br>20  <br>21  <br>22  <br>23  <br>24  <br>25  <br>26  <br>27  <br>28  <br>29  <br>30  <br>31  <br>32  <br>33  <br>34  <br>35  <br>36 | auth = "certificate"  <br>  <br>  <br>  <br>max-clients = 16  <br>max-same-clients = 2  <br>  <br>  <br>tcp-port = 1234  <br>udp-port = 1234  <br>  <br>  <br>  <br>  <br>  <br>mobile-dpd = 1800  <br>  <br>  <br>try-mtu-discovery = true  <br>  <br>  <br>server-cert = /etc/ocserv/pki/server/server.crt  <br>server-key = /etc/ocserv/pki/server/server.key  <br>  <br>  <br>ca-cert = /etc/ocserv/pki/ca/ca.crt  <br>  <br>  <br>ipv4-network = 192.168.101.0  <br>ipv4-netmask = 255.255.255.0  <br>  <br>  <br>dns = 8.8.8.8  <br>dns = 8.8.4.4 |

#### 配置证书

创建目录

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3 | mkdir /etc/ocserv/pki && cd /etc/ocserv/pki`  <br>  <br>mkdir server ca clients template |

配置 Server 证书

bash

|     |     |
| --- | --- |
| 1  <br>2 | mkdir /etc/ocserv/pki && cd /etc/ocserv/pki  <br>mkdir server ca clients template |

配置 CA 证书

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7  <br>8  <br>9  <br>10  <br>11  <br>12  <br>13  <br>14  <br>15  <br>16 | cd ../ca  <br>certtool --generate-privkey --sec-param high --outfile ca.key  <br>  <br>cat &lt;< \_EOF\_ &gt;../template/ca.tmpl  <br>cn = "VPN CA"  <br>organization = "Mid-south Sea"  <br>serial = 1  <br>expiration_days = 9999  <br>ca  <br>signing_key  <br>cert\_signing\_key  <br>crl\_signing\_key  <br>\_EOF\_  <br>  <br>certtool --generate-self-signed --load-privkey ca.key --template ../template/ca.tmpl --outfile ca.crt  <br>chmod 400 ca.key |

配置 Client 证书

bash

|     |     |
| --- | --- |
| 1  <br>2 | cd ../template  <br>vim client.tmpl |

输入以下内容（可自己随意修改）

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7  <br>8  <br>9  <br>10  <br>11  <br>12  <br>13 | cn = user  <br>o = "Organization"  <br>email = user@example.com  <br>dns_name = "www.example.com"  <br>country = US  <br>state = "New York"  <br>serial = 1  <br>expiration_days = 9999  <br>signing_key  <br>encryption_key   <br>tls\_www\_client  <br>ipsec\_ike\_key  <br>time\_stamping\_key |

制作自动签发脚本

bash

|     |     |
| --- | --- |
| 1  <br>2 | cd ..  <br>vim make-client.sh |

输入以下内容

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7  <br>8  <br>9  <br>10  <br>11  <br>12 | #!/bin/sh  <br>serial=\`date +%s\`  <br>certtool --generate-privkey --outfile clients/$1.key  <br>sed -i "1ccn = ${1}" template/client.tmpl  <br>sed -i "3cemail = ${1}@example.com" template/client.tmpl  <br>sed -i "7cserial = ${serial}" template/client.tmpl  <br>certtool --generate-certificate --load-privkey clients/$1.key --load-ca-certificate ca/ca.crt --load-ca-privkey ca/ca.key --template template/client.tmpl --outfile clients/$1.crt  <br>openssl pkcs12 -export -inkey clients/$1.key -in clients/$1.crt -name "$1 VPN Client Cert" -certfile ca/ca.crt -out clients/$1.p12  <br>exit 0  <br>  <br>  <br>chmod 700 make-client.sh |

然后就能用脚本很方便地生成客户端证书了：

bash

|     |     |
| --- | --- |
| 1   | ./make-client.sh testuser |

#### 启动 OCSERV 并设置开机启动

bash

|     |     |
| --- | --- |
| 1  <br>2 | systemctl start ocserv  <br>systemctl enable ocserv |

#### 配置 FIREWALLD

创建一个 ocserv 服务

bash

|     |     |
| --- | --- |
| 1   | vim /etc/firewalld/services/ocserv.xml |

内容如下：

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7 | <?xml version="1.0" encoding="utf-8"?>  <br>&lt;service&gt;  <br> &lt;short&gt;ocserv&lt;/short&gt;  <br> &lt;description&gt;Cisco AnyConnect&lt;/description&gt;  <br> <port protocol="tcp" port="1234"/>  <br> <port protocol="udp" port="1234"/>  <br>&lt;/service&gt; |

启动 firewalld

dockerfile

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4 | systemctl start firewalld  <br>firewall-cmd --permanent --add-service=ocserv  <br>firewall-cmd --permanent --add-masquerade  <br>firewall-cmd --reload |

#### 配置客户端

如果之前用 make-client.sh 生成过证书，那么在 / etc/ocserv/pki/client 目录下可以找到响应的 p12 文件。  
将该文件传到手机 / iPad 等终端上。

#### 提示

1.  虽然上面提到自签证书的信息可以随意写，但由于证书本身的信息是明文传输的，所以不要写奇怪的字段，以免引起防火墙注意。
2.  Server 证书不建议使用 ECC 证书，因为 AnyConnect 会提示不安全。CA 和 Client 证书不能使用 ECC 证书，因为 OpenConnect 不支持。

#### 路由表

bash

|     |     |
| --- | --- |
| 1  <br>2  <br>3  <br>4  <br>5  <br>6  <br>7  <br>8  <br>9  <br>10  <br>11  <br>12  <br>13  <br>14  <br>15  <br>16  <br>17  <br>18  <br>19  <br>20  <br>21  <br>22  <br>23  <br>24  <br>25  <br>26  <br>27  <br>28  <br>29  <br>30  <br>31  <br>32  <br>33  <br>34  <br>35  <br>36  <br>37  <br>38  <br>39  <br>40  <br>41  <br>42  <br>43  <br>44  <br>45  <br>46  <br>47  <br>48  <br>49  <br>50  <br>51  <br>52  <br>53  <br>54  <br>55  <br>56  <br>57  <br>58  <br>59  <br>60  <br>61  <br>62  <br>63  <br>64  <br>65  <br>66  <br>67  <br>68  <br>69  <br>70  <br>71  <br>72  <br>73  <br>74  <br>75  <br>76  <br>77  <br>78  <br>79  <br>80  <br>81  <br>82  <br>83  <br>84  <br>85  <br>86  <br>87  <br>88  <br>89  <br>90  <br>91  <br>92  <br>93  <br>94  <br>95  <br>96  <br>97  <br>98  <br>99  <br>100  <br>101  <br>102  <br>103  <br>104  <br>105  <br>106  <br>107  <br>108  <br>109  <br>110  <br>111  <br>112  <br>113  <br>114  <br>115  <br>116  <br>117  <br>118  <br>119  <br>120  <br>121  <br>122  <br>123  <br>124  <br>125  <br>126  <br>127  <br>128  <br>129  <br>130  <br>131  <br>132  <br>133  <br>134  <br>135  <br>136  <br>137  <br>138  <br>139  <br>140  <br>141  <br>142  <br>143  <br>144  <br>145  <br>146  <br>147  <br>148  <br>149  <br>150  <br>151  <br>152  <br>153  <br>154  <br>155  <br>156  <br>157  <br>158  <br>159  <br>160  <br>161  <br>162  <br>163  <br>164  <br>165  <br>166  <br>167  <br>168  <br>169  <br>170  <br>171  <br>172  <br>173  <br>174  <br>175  <br>176  <br>177  <br>178  <br>179  <br>180  <br>181  <br>182  <br>183  <br>184  <br>185  <br>186  <br>187  <br>188  <br>189  <br>190  <br>191  <br>192  <br>193  <br>194  <br>195  <br>196  <br>197  <br>198  <br>199 | no-route = 服务器 IP/255.255.255.255  <br>no-route = 1.0.0.0/255.192.0.0  <br>no-route = 1.64.0.0/255.224.0.0  <br>no-route = 1.112.0.0/255.248.0.0  <br>no-route = 1.176.0.0/255.240.0.0  <br>no-route = 1.192.0.0/255.240.0.0  <br>no-route = 14.0.0.0/255.224.0.0  <br>no-route = 14.96.0.0/255.224.0.0  <br>no-route = 14.128.0.0/255.224.0.0  <br>no-route = 14.192.0.0/255.224.0.0  <br>no-route = 27.0.0.0/255.192.0.0  <br>no-route = 27.96.0.0/255.224.0.0  <br>no-route = 27.128.0.0/255.224.0.0  <br>no-route = 27.176.0.0/255.240.0.0  <br>no-route = 27.192.0.0/255.224.0.0  <br>no-route = 27.224.0.0/255.252.0.0  <br>no-route = 36.0.0.0/255.192.0.0  <br>no-route = 36.96.0.0/255.224.0.0  <br>no-route = 36.128.0.0/255.192.0.0  <br>no-route = 36.192.0.0/255.224.0.0  <br>no-route = 36.240.0.0/255.240.0.0  <br>no-route = 39.0.0.0/255.255.0.0  <br>no-route = 39.64.0.0/255.224.0.0  <br>no-route = 39.96.0.0/255.240.0.0  <br>no-route = 39.128.0.0/255.192.0.0  <br>no-route = 40.72.0.0/255.254.0.0  <br>no-route = 40.125.128.0/255.255.128.0  <br>no-route = 40.126.64.0/255.255.192.0  <br>no-route = 42.0.0.0/255.248.0.0  <br>no-route = 42.48.0.0/255.240.0.0  <br>no-route = 42.80.0.0/255.240.0.0  <br>no-route = 42.96.0.0/255.224.0.0  <br>no-route = 42.128.0.0/255.128.0.0  <br>no-route = 43.224.0.0/255.224.0.0  <br>no-route = 45.64.0.0/255.255.128.0  <br>no-route = 45.112.0.0/255.240.0.0  <br>no-route = 47.92.0.0/255.252.0.0  <br>no-route = 47.96.0.0/255.224.0.0  <br>no-route = 49.0.0.0/255.248.0.0  <br>no-route = 49.48.0.0/255.248.0.0  <br>no-route = 49.64.0.0/255.224.0.0  <br>no-route = 49.112.0.0/255.240.0.0  <br>no-route = 49.128.0.0/255.224.0.0  <br>no-route = 49.208.0.0/255.240.0.0  <br>no-route = 49.224.0.0/255.224.0.0  <br>no-route = 52.80.0.0/255.252.0.0  <br>no-route = 54.222.0.0/255.254.0.0  <br>no-route = 58.0.0.0/255.128.0.0  <br>no-route = 58.128.0.0/255.224.0.0  <br>no-route = 58.192.0.0/255.224.0.0  <br>no-route = 58.240.0.0/255.240.0.0  <br>no-route = 59.32.0.0/255.224.0.0  <br>no-route = 59.64.0.0/255.224.0.0  <br>no-route = 59.96.0.0/255.240.0.0  <br>no-route = 59.144.0.0/255.240.0.0  <br>no-route = 59.160.0.0/255.224.0.0  <br>no-route = 59.192.0.0/255.192.0.0  <br>no-route = 60.0.0.0/255.224.0.0  <br>no-route = 60.48.0.0/255.240.0.0  <br>no-route = 60.160.0.0/255.224.0.0  <br>no-route = 60.192.0.0/255.192.0.0  <br>no-route = 61.0.0.0/255.192.0.0  <br>no-route = 61.80.0.0/255.248.0.0  <br>no-route = 61.128.0.0/255.192.0.0  <br>no-route = 61.224.0.0/255.224.0.0  <br>no-route = 91.234.36.0/255.255.255.0  <br>no-route = 101.0.0.0/255.128.0.0  <br>no-route = 101.128.0.0/255.224.0.0  <br>no-route = 101.192.0.0/255.240.0.0  <br>no-route = 101.224.0.0/255.224.0.0  <br>no-route = 103.0.0.0/255.192.0.0  <br>no-route = 103.192.0.0/255.240.0.0  <br>no-route = 103.224.0.0/255.224.0.0  <br>no-route = 106.0.0.0/255.128.0.0  <br>no-route = 106.224.0.0/255.240.0.0  <br>no-route = 110.0.0.0/255.128.0.0  <br>no-route = 110.144.0.0/255.240.0.0  <br>no-route = 110.160.0.0/255.224.0.0  <br>no-route = 110.192.0.0/255.192.0.0  <br>no-route = 111.0.0.0/255.192.0.0  <br>no-route = 111.64.0.0/255.224.0.0  <br>no-route = 111.112.0.0/255.240.0.0  <br>no-route = 111.128.0.0/255.192.0.0  <br>no-route = 111.192.0.0/255.224.0.0  <br>no-route = 111.224.0.0/255.240.0.0  <br>no-route = 112.0.0.0/255.128.0.0  <br>no-route = 112.128.0.0/255.240.0.0  <br>no-route = 112.192.0.0/255.252.0.0  <br>no-route = 112.224.0.0/255.224.0.0  <br>no-route = 113.0.0.0/255.128.0.0  <br>no-route = 113.128.0.0/255.240.0.0  <br>no-route = 113.192.0.0/255.192.0.0  <br>no-route = 114.16.0.0/255.240.0.0  <br>no-route = 114.48.0.0/255.240.0.0  <br>no-route = 114.64.0.0/255.192.0.0  <br>no-route = 114.128.0.0/255.240.0.0  <br>no-route = 114.192.0.0/255.192.0.0  <br>no-route = 115.0.0.0/255.0.0.0  <br>no-route = 116.0.0.0/255.0.0.0  <br>no-route = 117.0.0.0/255.128.0.0  <br>no-route = 117.128.0.0/255.192.0.0  <br>no-route = 118.16.0.0/255.240.0.0  <br>no-route = 118.64.0.0/255.192.0.0  <br>no-route = 118.128.0.0/255.128.0.0  <br>no-route = 119.0.0.0/255.128.0.0  <br>no-route = 119.128.0.0/255.192.0.0  <br>no-route = 119.224.0.0/255.224.0.0  <br>no-route = 120.0.0.0/255.192.0.0  <br>no-route = 120.64.0.0/255.224.0.0  <br>no-route = 120.128.0.0/255.240.0.0  <br>no-route = 120.192.0.0/255.192.0.0  <br>no-route = 121.0.0.0/255.128.0.0  <br>no-route = 121.192.0.0/255.192.0.0  <br>no-route = 122.0.0.0/254.0.0.0  <br>no-route = 124.0.0.0/255.0.0.0  <br>no-route = 125.0.0.0/255.128.0.0  <br>no-route = 125.160.0.0/255.224.0.0  <br>no-route = 125.192.0.0/255.192.0.0  <br>no-route = 137.59.88.0/255.255.252.0  <br>no-route = 139.0.0.0/255.224.0.0  <br>no-route = 139.128.0.0/255.128.0.0  <br>no-route = 140.64.0.0/255.240.0.0  <br>no-route = 140.128.0.0/255.240.0.0  <br>no-route = 140.192.0.0/255.192.0.0  <br>no-route = 144.0.0.0/255.255.0.0  <br>no-route = 144.7.0.0/255.255.0.0  <br>no-route = 144.12.0.0/255.255.0.0  <br>no-route = 144.52.0.0/255.255.0.0  <br>no-route = 144.123.0.0/255.255.0.0  <br>no-route = 144.255.0.0/255.255.0.0  <br>no-route = 150.0.0.0/255.255.0.0  <br>no-route = 150.96.0.0/255.224.0.0  <br>no-route = 150.128.0.0/255.240.0.0  <br>no-route = 150.192.0.0/255.192.0.0  <br>no-route = 152.104.128.0/255.255.128.0  <br>no-route = 153.0.0.0/255.192.0.0  <br>no-route = 153.96.0.0/255.224.0.0  <br>no-route = 157.0.0.0/255.255.0.0  <br>no-route = 157.18.0.0/255.255.0.0  <br>no-route = 157.61.0.0/255.255.0.0  <br>no-route = 157.122.0.0/255.255.0.0  <br>no-route = 157.148.0.0/255.255.0.0  <br>no-route = 157.156.0.0/255.255.0.0  <br>no-route = 157.255.0.0/255.255.0.0  <br>no-route = 159.226.0.0/255.255.0.0  <br>no-route = 161.207.0.0/255.255.0.0  <br>no-route = 162.105.0.0/255.255.0.0  <br>no-route = 163.0.0.0/255.192.0.0  <br>no-route = 163.96.0.0/255.224.0.0  <br>no-route = 163.128.0.0/255.192.0.0  <br>no-route = 163.192.0.0/255.224.0.0  <br>no-route = 166.111.0.0/255.255.0.0  <br>no-route = 167.139.0.0/255.255.0.0  <br>no-route = 167.189.0.0/255.255.0.0  <br>no-route = 167.220.244.0/255.255.252.0  <br>no-route = 168.160.0.0/255.255.0.0  <br>no-route = 171.0.0.0/255.128.0.0  <br>no-route = 171.192.0.0/255.224.0.0  <br>no-route = 175.0.0.0/255.128.0.0  <br>no-route = 175.128.0.0/255.192.0.0  <br>no-route = 180.64.0.0/255.192.0.0  <br>no-route = 180.128.0.0/255.128.0.0  <br>no-route = 182.0.0.0/255.0.0.0  <br>no-route = 183.0.0.0/255.192.0.0  <br>no-route = 183.64.0.0/255.224.0.0  <br>no-route = 183.128.0.0/255.128.0.0  <br>no-route = 192.124.154.0/255.255.255.0  <br>no-route = 192.188.170.0/255.255.255.0  <br>no-route = 202.0.0.0/255.128.0.0  <br>no-route = 202.128.0.0/255.192.0.0  <br>no-route = 202.192.0.0/255.224.0.0  <br>no-route = 203.0.0.0/255.128.0.0  <br>no-route = 203.128.0.0/255.192.0.0  <br>no-route = 203.192.0.0/255.224.0.0  <br>no-route = 210.0.0.0/255.192.0.0  <br>no-route = 210.64.0.0/255.224.0.0  <br>no-route = 210.160.0.0/255.224.0.0  <br>no-route = 210.192.0.0/255.224.0.0  <br>no-route = 211.64.0.0/255.248.0.0  <br>no-route = 211.80.0.0/255.240.0.0  <br>no-route = 211.96.0.0/255.248.0.0  <br>no-route = 211.136.0.0/255.248.0.0  <br>no-route = 211.144.0.0/255.240.0.0  <br>no-route = 211.160.0.0/255.248.0.0  <br>no-route = 218.0.0.0/255.128.0.0  <br>no-route = 218.160.0.0/255.224.0.0  <br>no-route = 218.192.0.0/255.192.0.0  <br>no-route = 219.64.0.0/255.224.0.0  <br>no-route = 219.128.0.0/255.224.0.0  <br>no-route = 219.192.0.0/255.192.0.0  <br>no-route = 220.96.0.0/255.224.0.0  <br>no-route = 220.128.0.0/255.128.0.0  <br>no-route = 221.0.0.0/255.224.0.0  <br>no-route = 221.96.0.0/255.224.0.0  <br>no-route = 221.128.0.0/255.128.0.0  <br>no-route = 222.0.0.0/255.0.0.0  <br>no-route = 223.0.0.0/255.224.0.0  <br>no-route = 223.64.0.0/255.192.0.0  <br>no-route = 223.128.0.0/255.128.0.0 |

Copyright Notice: All articles in this blog are licensed under [BSD 3-Clause "New" or "Revised" License](https://opensource.org/licenses/BSD-3-Clause) unless stating additionally.