---
layout: post
title: iOS自动打包
date: 2017-07-02 18:32:24.000000000 +09:00
categories: 
- iOS


tags:
- 脚本
---

本文首发于[个人博客](https://ityongzhen.github.io/KVC%E9%82%A3%E7%82%B9%E5%84%BF%E4%BA%8B.html)
## 前言
通过之前[Shell脚本](https://ityongzhen.github.io/categories/Shell/)的学习,整理iOS自动打包脚本

~~~~
#!/bin/sh

# 使用方法:
# step1: 将该脚本放在工程的根目录下（跟.xcworkspace文件or .xcodeproj文件同目录）
# step2: 根据情况修改下面的参数
# step3: 打开终端，执行脚本。（输入sh ，然后将脚本文件拉到终端，会生成文件路径，然后enter就可）

# =============项目自定义部分(自定义好下列参数后再执行该脚本)=================== #

# 是否编译工作空间 (例:若是用Cocopods管理的.xcworkspace项目,赋值true;用Xcode默认创建的.xcodeproj,赋值false)
is_workspace="true"

# .xcworkspace的名字，如果is_workspace为true，则必须填。否则可不填
workspace_name="xcworkspace的名字"

# .xcodeproj的名字，如果is_workspace为false，则必须填。否则可不填
project_name="xcworkspace的名字"

# 指定项目的scheme名称（也就是工程的target名称），必填
scheme_name="xcworkspace的名字"

# 指定要打包编译的方式 : Release,Debug。一般用Release。必填
build_configuration="Release"


# method，打包的方式。方式分别为 development, ad-hoc, app-store, enterprise 。必填
echo "Place enter the number you want to export ? \033[33;1m [1:app-store 2:ad-hoc 3:development 4:enterprise 5:gitlog]\033[0m"

read number
while([[ $number != 1 ]] && [[ $number != 2 ]] && [[ $number != 3 ]] && [[ $number != 4 ]] && [[ $number != 5 ]])
do
echo "Place enter the number you want to export ? \033[33;1m [1:app-store 2:ad-hoc 3:development 4:enterprise 5:gitlog]\033[0m"
read number
done

if [ $number == 1 ];then
method="app-store"
fi

if [ $number == 2 ];then
method="ad-hoc"
fi

if [ $number == 3 ];then
method="development"
fi

if [ $number == 4 ];then
method="development"
fi




#  下面两个参数只是在手动指定Pofile文件的时候用到，如果使用Xcode自动管理Profile,直接留空就好
# (跟method对应的)mobileprovision文件名，需要先双击安装.mobileprovision文件.手动管理Profile时必填
mobileprovision_name=""

# 项目的bundleID，手动管理Profile时必填
bundle_identifier=""


echo "--------------------脚本配置参数检查--------------------"
echo "\033[33;1mis_workspace=${is_workspace} "
echo "workspace_name=${workspace_name}"
echo "project_name=${project_name}"
echo "scheme_name=${scheme_name}"
echo "build_configuration=${build_configuration}"
echo "bundle_identifier=${bundle_identifier}"
echo "method=${method}"
echo "mobileprovision_name=${mobileprovision_name} \033[0m"


# =======================脚本的一些固定参数定义(无特殊情况不用修改)====================== #

# 获取当前脚本所在目录
script_dir="$( cd "$( dirname "$0"  )" && pwd  )"
# 工程根目录
project_dir=$script_dir

# 时间
DATE=`date '+%Y%m%d_%H%M%S'`
# 指定输出导出文件夹路径
export_path="$project_dir/Package/$scheme_name-$DATE"
# 指定输出归档文件路径
export_archive_path="$export_path/$scheme_name.xcarchive"
# 指定输出ipa文件夹路径
export_ipa_path="$export_path"
# 指定输出ipa名称
ipa_name="${scheme_name}_${DATE}"
# 指定导出ipa包需要用到的plist配置文件的路径
export_options_plist_path="$project_dir/ExportOptions.plist"

dSYM_path="$export_archive_path/dSYMs/$scheme_name.app.dSYM"
new_dSYM_path="$export_path/$scheme_name.app.dSYM"
zip_dSYM_path="$export_path/$scheme_name.app.dSYM.zip"

echo "--------------------脚本固定参数检查--------------------"
echo "\033[33;1mproject_dir=${project_dir}"
echo "DATE=${DATE}"
echo "export_path=${export_path}"
echo "export_archive_path=${export_archive_path}"
echo "export_ipa_path=${export_ipa_path}"
echo "export_options_plist_path=${export_options_plist_path}"
echo "ipa_name=${ipa_name}"
echo "zip_dSYM_path=${zip_dSYM_path} \033[0m"

# =======================自动打包部分(无特殊情况不用修改)====================== #

echo "------------------------------------------------------"
echo "\033[32m开始构建项目  \033[0m"
# 进入项目工程目录
cd ${project_dir}

# 指定输出文件目录不存在则创建
if [ -d "$export_path" ] ; then
    echo $export_path
else
    mkdir -pv $export_path
fi


if [ $number == 5 ];then

#git 日志路径
txt_path=$export_path/`date +%Y-%m-%d-%H_%M_%S.html`
temp_txt=$export_path/gitlog.html

#写入 ipa 下载链接
# echo "<div style='height: 40px;font-size: 20px;'>iOS 安装包下载链接：<a href=${ipa_link}>$ipa_link</a></div>" >> $txt_path 

#抓取 git 日志
git log --pretty=format:"<span style='font-size: 16px;'>%ai,%an:</span> <span style='color: #469cf7;font-size: 16px;'>%s</span><br>" \
 --no-merges  >> $txt_path

# 第二行插入 <div>
sed "2 i\ 
<div style='letter-spacing: 0.5px;line-height: 25px'>" $txt_path > $temp_txt
echo `cat $temp_txt` > $txt_path 
 
# 最后一行插入 </div>
sed "$ a\ 
</div>" $txt_path > $temp_txt
echo `cat $temp_txt` > $txt_path 

#移除中间转换文件
rm -f $temp_txt

#打开文件夹
open $export_path
open $txt_path

exit 0
fi


# 判断编译的项目类型是workspace还是project
if $is_workspace ; then
# 编译前清理工程
xcodebuild clean -workspace ${workspace_name}.xcworkspace \
                 -scheme ${scheme_name} \
                 -configuration ${build_configuration}

xcodebuild archive -workspace ${workspace_name}.xcworkspace \
                   -scheme ${scheme_name} \
                   -configuration ${build_configuration} \
                   -archivePath ${export_archive_path}
else
# 编译前清理工程
xcodebuild clean -project ${project_name}.xcodeproj \
                 -scheme ${scheme_name} \
                 -configuration ${build_configuration}

xcodebuild archive -project ${project_name}.xcodeproj \
                   -scheme ${scheme_name} \
                   -configuration ${build_configuration} \
                   -archivePath ${export_archive_path}
fi

#  检查是否构建成功
#  xcarchive 实际是一个文件夹不是一个文件所以使用 -d 判断
if [ -d "$export_archive_path" ] ; then
    echo "\033[32;1m项目构建成功 🚀 🚀 🚀  \033[0m"
else
    echo "\033[31;1m项目构建失败 😢 😢 😢  \033[0m"
    exit 1
fi
echo "------------------------------------------------------"

echo "\033[32m开始导出ipa文件 \033[0m"


# 先删除export_options_plist文件
if [ -f "$export_options_plist_path" ] ; then
    #echo "${export_options_plist_path}文件存在，进行删除"
    rm -f $export_options_plist_path
fi
# 根据参数生成export_options_plist文件
/usr/libexec/PlistBuddy -c  "Add :method String ${method}"  $export_options_plist_path
/usr/libexec/PlistBuddy -c  "Add :provisioningProfiles:"  $export_options_plist_path
/usr/libexec/PlistBuddy -c  "Add :provisioningProfiles:${bundle_identifier} String ${mobileprovision_name}"  $export_options_plist_path
/usr/libexec/PlistBuddy -c  "Add :compileBitcode bool NO"  $export_options_plist_path

xcodebuild  -exportArchive \
            -archivePath ${export_archive_path} \
            -exportPath ${export_ipa_path} \
            -exportOptionsPlist ${export_options_plist_path} \
            -allowProvisioningUpdates

# 检查ipa文件是否存在
if [ -f "$export_ipa_path/$scheme_name.ipa" ] ; then
    echo "\033[32;1mexportArchive ipa包成功,准备进行重命名\033[0m"
else
    echo "\033[31;1mexportArchive ipa包失败 😢 😢 😢     \033[0m"
    exit 1
fi

# 修改ipa文件名称
mv $export_ipa_path/$scheme_name.ipa $export_ipa_path/$ipa_name.ipa

# 检查文件是否存在
if [ -f "$export_ipa_path/$ipa_name.ipa" ] ; then
    echo "\033[32;1m导出 ${ipa_name}.ipa 包成功 🎉  🎉  🎉   \033[0m"
    # open $export_path
else
    echo "\033[31;1m导出 ${ipa_name}.ipa 包失败 😢 😢 😢     \033[0m"
    exit 1
fi

# 删除export_options_plist文件（中间文件）
if [ -f "$export_options_plist_path" ] ; then
    #echo "${export_options_plist_path}文件存在，准备删除"
    rm -f $export_options_plist_path
fi

#导出dSYM并压缩 zip
cp -r "${dSYM_path}" "${export_path}"
zip -r "$zip_dSYM_path" "$new_dSYM_path"
rm -r $new_dSYM_path

# 输出打包总用时
echo "\033[36;1m使用AutoPackage打包总用时: ${SECONDS}s \033[0m"

#准备上传dSYM文件到 bugly
echo "\033[33m ==========准备上传dSYM文件到 bugly============ \033[0m"

curl -k "https://api.bugly.qq.com/openapi/file/upload/symbol?app_key=4ec76824-0694-4de5-a0a3-447f3ad4259a&app_id=cf4e8e6eaa" \
--form "api_version=1" --form "app_id=cf4e8e6eaa" --form "app_key=4ec76824-0694-4de5-a0a3-447f3ad4259a" \
--form "symbolType=2" --form "bundleId=yourBundleId"  --form "fileName=$scheme_name.app.dSYM.zip" \
--form "file=@/$zip_dSYM_path" --verbose | python -m json.tool


if [ $number == 1 ];then
echo "\033[33m ==========准备上传ipa到App Store============ \033[0m"
#验证并上传到App Store，将-u 后面的XXX替换成自己的AppleID的账号，-p后面的XXX替换成自己的密码
altoolPath="/Applications/Xcode.app/Contents/Applications/Application Loader.app/Contents/Frameworks/ITunesSoftwareService.framework/Versions/A/Support/altool"
"$altoolPath" --validate-app -f ${ipa_name}.ipa -u appid账号 -p appid密码 -t ios --output-format xml
"$altoolPath" --upload-app -f ${ipa_name}.ipa -u  appid账号 -p appid密码 -t ios --output-format xml
else

echo "\033[33m ==========准备上传ipa到fir============ \033[0m"
#自定义短链
short="rxios"

#ipa 完整下载链接
ipa_link="https://fir.im/$short"

#上传 ipa
fir publish $export_ipa_path/$ipa_name.ipa -T '4596a0fbf2d0adeaa017302b17686f0c' -Q -s "$short"

#git 日志路径
txt_path=$export_path/`date +%Y-%m-%d-%H_%M_%S.html`
temp_txt=$export_path/gitlog.html

#写入 ipa 下载链接
echo "<div style='height: 40px;font-size: 20px;'>iOS 安装包下载链接：<a href=${ipa_link}>$ipa_link</a></div>" >> $txt_path 

#抓取 git 日志
git log --pretty=format:"<span style='font-size: 16px;'>%ai,%an:</span> <span style='color: #469cf7;font-size: 16px;'>%s</span><br>" \
 --no-merges  >> $txt_path

# 第二行插入 <div>
sed "2 i\ 
<div style='letter-spacing: 0.5px;line-height: 25px'>" $txt_path > $temp_txt
echo `cat $temp_txt` > $txt_path 
 
# 最后一行插入 </div>
sed "$ a\ 
</div>" $txt_path > $temp_txt
echo `cat $temp_txt` > $txt_path 

#发送版本信息到邮箱，为了防止在widndows电脑上文本乱码，邮件内容用html的形式
# mail_title="容信iOS安装包已更新"
# mail -s  "$(echo "${mail_title}\nContent-Type:text/html;charset=UTF-8")"  \
# eagle_01yz@163.com\
# < ${txt_path}


#移除中间转换文件
rm -f $temp_txt
fi

#打开文件夹
open $export_path
open $txt_path

exit 0


~~~~

