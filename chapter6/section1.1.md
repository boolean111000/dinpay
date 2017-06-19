## 常用IP限制功能


** 在我们实现网站的时候,为了防止其他人员访问我们的网站,可以在入口的第一关进行IP的验证.我们可以在根目录添加一个txt的黑名单/白名单,接收用户的ip,然后读取ip名单,判断是否在我们允许的范围内,否则直接退出程序,从而有效的保护我们的程序安全,也可以进行设置黑名单,对一些恶意攻击者进行防御. **

> 案例一

    #程序入口文件/var/www/mtcp/cpweb/index.php
    <?php
    @header("Content-Type:text/html; charset=utf-8");
    define("ROOT_PATH",realpath(dirname(__FILE__).'/../').DIRECTORY_SEPARATOR);
    define("IP_BLACKLIST_FILE",ROOT_PATH."ip_blacklist.txt");
    
    $client_ip = trim($_SERVER['REMOTE_ADDR']);
    $real_ip = trim($_SERVER["HTTP_X_FORWARDED_FOR"]);
    if(strpos($real_ip,',')){
    #获取代理网关如果有多级会用逗号来进行分隔,我们获取第一个作为原始的
    $real_ip = substr($real_ip,0,strpos($real_ip,','));
    $real_ip = strtolower($real_ip);
    }

    if(!function_exists('checkIP')){
        function checkIp($ip){
            $exit_str = "YOUR IP IS NOT ALLOWED {$ip} ";
            $exit_str .= "<a href='http://kf.com'>联系客服</a>";
            $ip = trim($ip);
            $ip_list = file(IP_BLACKLIST_FILE);
            foreach($ip_list as $k => $line){
                $ip_list[$k] = trim($line);
            }
            if(in_array($ip,$ip_list)){
                exit($exit_str);
                return false;
            }
            $ip_arr = explode(".",$ip);
            $ip_arr[3] = '*';
            $ip3 = implode('.',$ip_arr);
            if(in_array($ip3,$ip_list)){
                exit($exit_str);
                return false;
            }
            
            $ip_arr[2] = '*';
            $ip2 = implode('.',$ip_arr);
            if(in_array($ip2,$ip_list)){
                exit($exit_str);
                return false;
            }
    }
    }
    checkIp($client_ip);
    ?>
ip列表文件,每行一个ip,可进行通配,注意这个文件权限问题,否则无法进行设置,读写
       #ip列表文件 /var/www/mtcp/ip_blacklist.txt
       1.24.220.255
       117.136.68.255
       210.199.183.255
       210.196.239.80
       218.15.*.* 
       192.168.0.*
这样当一个用户ip在列表额时候我们可以进行对应的操作,白名单是反过来的,就是不存在则退出
我们访问入口最终得到
``` YOUR IP IS NOT ALLOWED 218.15.207.136 联系客服 ```


> 案例二
    
    #入口文件index.php,判断是否在白名单里面
    <?php
    @header("Content-Type:text/html;charset=utf-8");
    #判断访问者ip,如果不在白名单里面禁止访问
    $accessIp = trim($_SERVER['REMOTE_ADDR']);
    $ipAllow = file('./framework/ipallow.txt');
    foreach ($ipAllow as $k => $line) {
        $ipAllow[$k] = trim($line);
    }
    $str = "您的IP[{$accessIp}],不在允许范围内,请联系管理员";
    if (!in_array($accessIp, $ipAllow)) {
        exit($str);
    }
    
    #set.php设置文件
    #设置白名单,第一遍是将我们白名单列表中的数据读取出来,显示到模板文件红,第二个四接收修改
    if(isset($_GET['val']) && ($_GET['val']=='iplist')){
         $iplist =  file_get_contents('./framework/ipallow.txt');
         include_once './view/ipaddress_edit.phtml';
    }
    #如果是修改的话我们直接进行写入操作这里注意文件权限
    if(isset($_POST['op']) && ($_POST['op']=='submit')){
            $content = $_POST['data'];
            if(strlen($content) > 0){
            file_put_contents('./framework/ipallow.txt',$content);
                echo json_encode(['errno'=>8,'errstr'=>'操作成功,已保存']);
                exit();
            }
    }
    
    #还有一个就是模板文件,进行提交设置
    #ipaddress_edit.phtml   
    <body>
    <div style="border:1px solid #cfd4e2;font-size:14px;background-color:#ebebeb;margin:5px;padding: 10px;font-weight:700;">
    <span>当前位置: ->编辑白名单</span></div>
    <div class="main" >
            <div class="title">编辑访问白名单</div>
            <textarea name="content" id="content" rows="30" cols="60">
            <?php echo $iplist; ?>
            </textarea>
            <div class="footer">
                <!--<input type="hidden" name="action" value="submit"/>-->
                <input class="sb" type="submit"  id="submit" value="提交" />
            </div>
        
    </div>
    </body>
    <script>
        function changeColor(x){
            x.style.borderColor = '#2689e4';
        }
        function resets(x){
            x.style.borderColor = '#bfcdda';
        }
        $(function(){
           $("#submit").click(function(){
               $.post(
             './assict.php',
            {
            op:'submit',
            data:$("#content").val()
            },
            function(response){
                if(response.errno === 8){
                    alert(response.errstr);
                }
                location.reload(true);
            },'json'
                     );
           });
            
        });
    </script>

