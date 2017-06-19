## IP定位API接入指南

> # 案例一

首先我们获取用户的真实IP,可以通过几种办法来实现,但是有的办法还是获取不到,所以我们最好做一个函数,进行获取

    #获取用户真实iP地址
    function Getip() {
    if (!empty($_SERVER["HTTP_CLIENT_IP"])) {
        $ip = $_SERVER["HTTP_CLIENT_IP"];
    }
    if (!empty($_SERVER['HTTP_X_FORWARDED_FOR'])) { //获取代理ip
        $ips = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
    }
    if ($ip) {
        $ips = array_unshift($ips, $ip);
    }
    
    $count = count($ips);
    for ($i = 0; $i < $count; $i++) {
        if (!preg_match("/^(10|172\.16|192\.168)\./i", $ips[$i])) {//排除局域网ip
            $ip = $ips[$i];
            break;
        }
    }
    $tip = empty($_SERVER['REMOTE_ADDR']) ? $ip : $_SERVER['REMOTE_ADDR'];
    return $tip;
    
    }
    
    #将ip带入到网络源进行查询
    #获取访客ip地点
    function getAddr($ip){
        $ipstr = trim($ip);
        $url="http://ip.taobao.com/service/getIpInfo.php?ip=".$ipstr; 
        $ipinfo = file_get_contents($url);
        $ipinfo = json_decode($ipinfo,true);
        $ipinfo = $ipinfo['data'];
        $str = $ipstr."[".$ipinfo['country'].'-'.$ipinfo['country'];
        $str .= ']-['.$ipinfo['country'].$ipinfo['isp'].']';
        #return $str;
        return $ipinfo;
    }
    #这个函数在组合的时候我们要进行修改,这个貌似是会变的,我们可以先打印出这个返回的信息
    #比如格式返回如下:
    Array
    (
    [country] => 中国
    [country_id] => CN
    [area] => 华东
    [area_id] => 300000
    [region] => 江苏省
    [region_id] => 320000
    [city] => 
    [city_id] => -1
    [county] => 
    [county_id] => -1
    [isp] => 移动
    [isp_id] => 100025
    [ip] => 117.136.67.199
    )
    
    Array
    (
    [country] => 台湾
    [country_id] => TW
    [area] => 
    [area_id] => 
    [region] => 台湾省
    [region_id] => TW_01
    [city] => 
    [city_id] => 
    [county] => 
    [county_id] => 
    [isp] => Hurricane Electric
    [isp_id] => 2000206
    [ip] => 111.184.0.175
    )
    Array
    (
    [country] => 中国
    [country_id] => CN
    [area] => 华中
    [area_id] => 400000
    [region] => 河南省
    [region_id] => 410000
    [city] => 郑州市
    [city_id] => 410100
    [county] => 登封市
    [county_id] => 410185
    [isp] => 电信
    [isp_id] => 100017
    [ip] => 171.10.197.12
    )

     #使用方法:
    $ipinfo = getAddr(Getip());
    打印之后得到组织好的数据
    171.10.197.12[中国河南省郑州市]-[中国电信]
    106.184.0.175[日本]-[日本]
    192.126.119.221[美国]-[美国]
    210.209.89.165[香港香港特别行政区]-[香港NEWWORLDTEL]

* 这个方法的坏处是,速度比较慢,网络不好的时候可能要很久,而且这个api会根据淘宝而变化,但是淘宝的覆盖面广
* 其他的IP API还有:
* http://freeapi.ipip.net/23.88.2.3 免费的,专业版的要收费的
* http://lbsyun.baidu.com/index.php?title=webapi/ip-api 百度的API也是可以的,但是接入需要一点时间
* 还有纯真IP数据库,hz的后台就是使用的这里的数据库,但是不知道怎么接入的,我们下面先来了解一下hz的ip实现原理



> # 案例二

纯真ip数据库对接,这个覆盖面比较大,可存储在本地上.他使用一个QQWry.DAT文件就可以搞定,一次查询时间快,只需要4ms,我们首先[下载纯真ip客户端](http://cz88.net)
安装软件之后,我们进入到安装目录,会看到一个qqwry.dat,这个就是我们的主角了,当然有的人也会使用软件的解压成txt来进行解析,存放到指定的目录,
[本地下载170119最新版](../download/qqwry.dat)
下面是php版的一个demo,测试有效,后面的数据可以根据自己的需要进行变动
* 首先到http://www.cz88.net/down/ 下载之后安装软件,在桌面右键打开所在目录,复制qqwry.dat到服务器的/var/www/mtcp/cpweb/
* 建立一个类文件/var/www/mtcp/cpweb/iplocation.class.php,内容如下



    <?php
    class IP {
    var $fh;        //IP数据库文件句柄
    var $first;        //第一条索引
    var $last;        //最后一条索引
    var $total;        //索引总数
    
    //构造函数
    function __construct(){
        $this->fh = fopen('qqwry.dat', 'rb');                //qqwry.dat文件,请指定路径
        $this->first = $this->getLong4();
        $this->last  = $this->getLong4();
        $this->total = ($this->last - $this->first) / 7;    //每条索引7字节
    }

    //检查IP合法性
    function checkIp($ip){
        $arr = explode('.', $ip);
        if( count($arr) !=4 ){
            return false;
        }
        else{
            for( $i=0; $i<4; $i++){
                if( $arr[$i] <'0' || $arr[$i] > '255' ) {
                    return false;
                }
            }
        }
        return true;
    }
    
    function getLong4(){
        //读取little-endian编码的4个字节转化为长整型数
        $result = unpack('Vlong', fread($this->fh, 4));
        return $result['long'];
    }
    
    function getLong3() {
        //读取little-endian编码的3个字节转化为长整型数
        $result = unpack('Vlong', fread($this->fh, 3).chr(0));
        return $result['long'];
    }

    //查询信息
    function getInfo($data = "") {
        $char = fread($this->fh, 1);
        while( ord($char) != 0) {        //国家地区信息以0结束
            $data .= $char;
            $char = fread($this->fh, 1);
        }
        return $data;
    }
    //查询地区信息
    function getArea() {
        $byte = fread($this->fh, 1);    //标志字节
        switch (ord($byte)) {
            case 0: $area = ''; break;    //没有地区信息
            case 1:        //地区被重定向
                fseek($this->fh, $this->getLong3());
                $area = $this->getInfo(); break;
            case 2:        //地区被重定向
            fseek($this->fh, $this->getLong3());
            $area = $this->getInfo(); break;
            default: $area = $this->getInfo($byte);  break;        //地区没有被重定向
        }
        return $area;
    }

    function ip2addr($ip) {
        if( !$this->checkIp($ip) ){
            return false;
        }

        $ip = pack('N', intval(ip2long($ip)) );

        //二分查找
        $l = 0;
        $r = $this->total;

        while($l <= $r) {
            $m = floor(($l + $r) / 2);                //计算中间索引
            fseek($this->fh, $this->first + $m * 7);
            $beginip = strrev(fread($this->fh, 4)); //中间索引的开始IP地址
            fseek($this->fh, $this->getLong3());
            $endip = strrev(fread($this->fh, 4));    //中间索引的结束IP地址

            if( $ip < $beginip) {    //用户的IP小于中间索引的开始IP地址时
                $r = $m - 1;
            }
            else{
                if( $ip > $endip) {    //用户的IP大于中间索引的结束IP地址时
                    $l = $m + 1;
                }
                else{                //用户IP在中间索引的IP范围内时
                    $findip = $this->first + $m * 7;
                    break;
                }
            }
        }

        //查询国家地区信息
        fseek($this->fh, $findip);
        $location['beginip'] = long2ip($this->getLong4());    //用户IP所在范围的开始地址
        $offset = $this->getlong3();
        fseek($this->fh, $offset);
        $location['endip'] = long2ip($this->getLong4());    //用户IP所在范围的结束地址
        
        $byte = fread($this->fh, 1);                //标志字节
        switch( ord($byte) ){
            case 1:            //国家和区域信息都被重定向
                $countryOffset = $this->getLong3();    //重定向地址
                fseek($this->fh, $countryOffset);
                $byte = fread($this->fh, 1);        //标志字节
                switch( ord($byte) ){
                    case 2: //国家信息被二次重定向
                        fseek($this->fh, $this->getLong3());
                        $location['country'] = $this->getInfo();
                        fseek($this->fh, $countryOffset + 4);
                        $location['area'] = $this->getArea();
                        break;
                    default: //国家信息没有被二次重定向
                        $location['country'] = $this->getInfo($byte);
                        $location['area'] = $this->getArea();
                        break;
                }
                break;

            case 2:            //国家信息被重定向
                fseek($this->fh, $this->getLong3());
                $location['country'] = $this->getInfo();
                fseek($this->fh, $offset + 8);
                $location['area'] = $this->getArea();
                break;

            default:        //国家信息没有被重定向
                $location['country'] = $this->getInfo($byte);
                $location['area'] = $this->getArea();
                break;
        }

        //gb2312 to utf-8（去除无信息时显示的CZ88.NET）
        foreach ($location as $k => $v) {
            $location[$k] = str_replace('CZ88.NET', '', iconv('gb2312', 'utf-8', $v));
        }

        return $location;
    }

    //析构函数
    function __destruct() {
        fclose($this->fh);
    }
    
    }


##### 在需要使用的页面载入或者是使用自动加载类载入这个类库实例化,传入ip,得到一个返回的数组,
##### 然后组织数据后就可以得到我们想要的数据地址了

> 返回数据格式如下:
Array
(
    [beginip] => 210.209.64.0
    [endip] => 210.209.127.255
    [country] => 香港
    [area] => 新世界电讯(NWT)数据中心
)
Array
(
    [beginip] => 192.126.112.0
    [endip] => 192.126.127.255
    [country] => 美国
    [area] => NexteCloud数据中心
)

    #/var/www/mtcp/cpweb/index.php
    require_once 'iplocation.class.php';
    $ip = new IP();
    $addr = $ip->ip2addr('192.126.119.221');
    #print_r($addr)返回上面的数组格式;
    echo '您好，您当前的IP所在地是:'.$addr['country'].$addr['area'];
    
> ###### 最终结果
    Array
    (
    [beginip] => 117.136.67.0
    [endip] => 117.136.67.255
    [country] => 江苏省常州市
    [area] => 移动GSM/TD-SCDMA/LTE全省共用出口
    )
    您好，您当前的IP所在地是:江苏省常州市移动GSM/TD-SCDMA/LTE全省共用出口
    




    