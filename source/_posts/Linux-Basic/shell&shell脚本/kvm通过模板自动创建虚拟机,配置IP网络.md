---
title: kvm自动创建虚拟机,自定义IP地址 
date: 2020-06-30 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---



```shell
#!/bin/bash
#通过KVM模板自动化创建虚拟机（需要电脑中存在模板）.以及创建完虚拟机后,自动修改IP地址
#该脚本需要以root身份执行


 #镜像和模板源文件
src_img_path=/opt/vmx/linux/hadoop.dev.base.img
src_xml_path=/etc/libvirt/qemu/hadoop.dev.base.xml

#镜像和模板源文件父目录
prefix_img_path=/opt/vmx/linux
prefix_xml_path=/etc/libvirt/qemu

#获取新虚拟机名称
get_newname(){
	while true
        do
                read -p "请输入新虚拟机名称(主机名)：" newname
                if [ $newname ];then
                    if `virsh list --all | grep "\b${newname}\b"`;then
                        echo "该虚拟机已经存在.请检查系统当前虚拟机"
                    else

                        #设置新虚拟机镜像和模板路径
                        new_img_path=${prefix_img_path}/${newname}.qcow2
                        new_xml_path=${prefix_xml_path}/${newname}.xml
                        break
                    fi
                else
                        echo "************"
                        echo "请输入虚拟机主机名！"
                        echo "************"
                fi
        done
}
#设置新虚拟机内存
get_newmemary(){
	while true
        do
                current_free_mem=`free -g|awk '/^Mem/{print $4}'`
                mem_total=`free -g|awk '/^Mem/{print $2}'`
                echo "目前本机内存总大小：${mem_total}g"
                echo "当前空闲内存大小为：${current_free_mem}g"
                read -p "请输入新虚拟机内存大小(单位G)：" newmemary
                if [ $newmemary ];then
                        if [[ $newmemary < $mem_total ]];then
                                break
                        else
                                echo "**********************************"
                                echo "输入的数值必须小于当前内存总大小！"
                                echo "**********************************"
                        fi
                else
                        echo "********************"
                        echo "请输入新虚拟机内存！"
                        echo "********************"
                fi
        done
}
#设置新虚拟机CPU
get_newcpu(){
	while true
        do
                core=`cat /proc/cpuinfo| grep "processor"| wc -l`
                echo "可用core个数：${core}"
                read -p "请输入新虚拟机处理器核数：" newcpu
                if [ $newcpu ];then
                        if [ $newcpu -le $core ];then
                                break
                        else
                                echo "******************************"
                                echo "不能超过可用个数或者输入错误！"
                                echo "******************************"
                        fi
                else
                        echo "**************"
                        echo "输入不能为空！"
                        echo "**************"
                fi
        done
}

#设置IP
get_ip(){

    while true
        do 
            read -p "请输入新虚拟机的IP地址(格式:172.16.10.[2-240]):" ip
            if `echo $ip| grep -E "172.16.10.[0-9]{1,3}$" > /dev/null 2>&1`;then
                ipaddr=$(echo $ip | awk -F "." '{print $4}')
                #valid_check=$(echo $ip | awk -F "." '2<$4 && $4<241 {print "yes"}')
                #if [ "$valid_check" == "yes" ];then
                if [ $ipaddr -gt 2 -a $ipaddr -lt 241 ];then
                    if ! `ping -W 3 -c 1 $ip > /dev/null 2>&1`;then
                       break
                    
                    else
                       echo "当前IP已经被占用,请重新输入"
                    
                    fi
                
                else
                    echo "IP地址必须要在2-240之间"

                fi
            
            else
                echo "IP地址格式不对,请重新输入"
            
            fi
    done
  

}

#复制模板、xml
copy_model_xml(){
	cp $src_img_path $new_img_path
        cp $src_xml_path $new_xml_path
}
#修改xml文件
modification_xml(){
    uuid=`uuidgen`
	sed -ri "s/(<name>).*(<\/name>)/\1${newname}\2/" $new_xml_path  
        sed -ri "s/(<uuid>).*(<\/uuid>)/\1${uuid}\2/" $new_xml_path 
        mem_kb=$((${newmemary}*1024*1024)) 
 
        sed -ri "s/(<memory.*>).*(<\/memory>)/\1${mem_kb}\2/" $new_xml_path 
        sed -ri "s/(<currentMemory.*>).*(<\/currentMemory>)/\1${mem_kb}\2/" $new_xml_path
        sed -ri "s/(<vcpu.*>).*(<\/vcpu>)/\1${newcpu}\2/" $new_xml_path
        sed -ri "s@(<source file=').*('\/>)@\1${new_img_path}\2@" $new_xml_path  #定义新虚拟机的xml文件路径

        sed -i "s@port='5930'@port='"59${ipaddr}"'@" $new_xml_path #配置VNC端口号.格式59+ip地址最后一位
        mac_addr=`openssl rand -hex 3 | sed -r 's/..\B/&:/g'` #生成一个随机的MAC地址
 
        sed -ri "s/(<mac address='..:..:..:).*('\/>)/\1${mac_addr}\2/" $new_xml_path #配置MAC地址
}
#define
define_vm(){
	virsh define $new_xml_path #从xml文件中生成新虚拟机
	echo "**********"
	echo "${newname}建完成！"
	echo "**********"
}

network_setting(){

    #该脚本使用guestmount工具，Centos7中安装libguestfs-tools-c可以获得guestmount工具
    #脚本在不登陆虚拟机的情况下，修改虚拟机的IP地址信息

    mountpath="/tmp/${newname}"
    [ ! -d $mountpath ] && mkdir $mountpath

    guestmount  -d ${newname} -i $mountpath

    #修改IP地址: 注意这里要通过这种方式才能修改IP..不能使用sed命令直接修改原文件,否则虚拟机启动后无法识别ifcfg-eth0文件
    cat > ${mountpath}/etc/sysconfig/network-scripts/ifcfg-eth0 << EOF
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=$ip
NETMASK=255.255.255.0
GATEWAY=172.16.10.254
DNS1=114.114.114.114
EOF
    #sed -i "/IPADDR/s@.*@IPADDR=${ip}@" ${mountpath}/etc/sysconfig/network-scripts/ifcfg-eth0
    #修改主机名
    echo "${newname}" > ${mountpath}/etc/hostname

    echo "虚拟主机IP地址和主机名配置完成"

    #卸载临时挂载目录
    umount /tmp/${newname}
    
    #删除临时挂载目录
    rm -rf /tmp/${newname}
    
}
#------------运行分界线------------------------------------

get_newname
get_newmemary
get_newcpu
get_ip
copy_model_xml
modification_xml
define_vm
network_setting
```

参考 https://blog.csdn.net/qq_41814635/article/details/82256970