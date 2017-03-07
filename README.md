# Run
#打包批处理


#!/bin/sh

#  run.sh
#  Quentin
#  自动编译ipa
#  Created by quentin on 16/5/30.
#  Copyright © 2016年 Quentin. All rights reserved.

#使用此脚本需要安装两个工具
#brew install ImageMagick
#brew install ghostscript

#  2016/06/15 不支持打APPStore的包
#  2016/06/16 添加DEV的icon图标转换
#  2016/06/22 支持打APPStore包并自动上传
#  2016/06/23 appstore用户名和密码外部传入以及图标的恢复并把info文件更改上传到svn

#  运行命令如下sh run.sh (appleUserName) (applePassword)打包appstore版本需要输入apple用户和密码，如果不输入在后续打包的时候会提醒

#编译条件
UAT=2
APPSTORE=3
DEV=1

userName=$1
password=$2

#工程路径
project_path=$(pwd)
project_name=$(ls | grep xcodeproj | awk -F.xcodeproj '{print $1}')

if [ "${project_name}" == "" ]
then
   echo "\033[31m 当前目录不是项目目录，不能进行编译打包 \033[0m"
exit
fi

if [ "${userName}" == "" -a "${password}" == "" ]
then
echo "\033[31m 如果打包为appStore可以使用命令:sh run.sh username password \033[0m"
fi

#打包工程
build_workspace=${project_name}".xcworkspace"

echo "################更新svn代码###################"

#这里根据自己项目修改，我们使用的为svn
svn_diff=$(svn di)

if [ "${svn_diff}" != "" ]
then

echo "\033[37m 本地代码有更改，是否恢复代码后再打包?改变如下：  \033[0m"
echo ${svn_diff}
select revert in "Y" "N"
do
case ${revert} in
"Y")
echo "恢复本地修改的代码"
svn revert -R .
break
;;
*)
echo "不恢复本地修改的代码"
break
;;
esac
done

fi
#svn up
echo "\033[37m 更新代码 \033[0m"
svn up
echo "\033[37m 更新完毕 \033[0m"
echo "################设置打包信息###################"

#此版本更新内容
echo "################此版本更新的内容###################"

for line in $(cat ${project_path}/UpdateContent)
do
echo ${line}
content=${content}" "${line}
done

echo
sleep 2

#选择打包类型
echo "请选择打包类型:" #这里根据自己项目修改
select type in "DEV" "UAT" "APPSTORE" "Exit"
do
case ${type} in
"DEV")
echo "您选择了DEV环境"
display_name="**" #这里改为自己项目的名字
GCC_PREPROCESSOR_DEFINITIONS=""
type=${DEV}
break
;;
"UAT")
echo "您选择了UAT环境"
display_name="**" #这里改为自己项目的名字
GCC_PREPROCESSOR_DEFINITIONS="GCC_PREPROCESSOR_DEFINITIONS"='"UAT=1"'#这里根据自己项目修改
type=${UAT}
break
;;
"APPSTORE")
echo "APPSTORE环境"
GCC_PREPROCESSOR_DEFINITIONS="GCC_PREPROCESSOR_DEFINITIONS"='"APPSTORE=1"'#这里根据自己项目修改
display_name="**" #这里改为自己项目的名字
type=${APPSTORE}
break
;;
"Exit")
echo "退出打包"
exit
;;
*)
echo "\033[31m 输入错误，请重新输入打包类型: \033[0m"
;;
esac
done;

#判断apple用户名和密码
echo
function appstore_username_password() {
if [ $1 == ${APPSTORE} ]
    then
    if [ "${userName}" == "" -o "${password}" == "" ]
    then
        echo "当前为AppStore版本，apple用户名或者密码为空，请重新输入:"
        echo "请输入apple用户名:"
        read userName
        echo "请输入apple密码:"
        read password
        echo "apple用户名:${userName} apple密码:${password}"
        appstore_username_password $1
    else
        echo "apple用户名和密码是否确认？(Y(确认)/N(重新输入)/Exit(退出打包))"
        select validate in "Y" "N" "Exit"
        do
        case ${validate} in
        "Y")
        break
        ;;
        "N")
        userName=""
        password=""
        appstore_username_password $1
        break
        ;;
        "Exit")
        exit
        ;;
        esac
        done
    fi
fi
}

appstore_username_password ${type}

#app info.plist
app_infoplist_path=${project_path}/${project_name}/Info.plist
if [ ${type} == ${DEV} ]
then
app_infoplist_path=${project_path}/${project_name}/**#这里根据自己项目修改
fi


#版本号
pre_version=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${app_infoplist_path}")

echo "上个版本号为:${pre_version}"
function version() {
    echo "请输入版本号(E.g:1.5.0):"
    read VERSION
    if [ "${VERSION}" == "" ]
    then
       echo "输入版本为空，重新输入"
       version
    else
        echo "\033[37m 版本号为:"${VERSION}" \033[0m"
    fi
}

echo
version
echo

#pod是否更新

#这里根据自己项目修改，我们使用了pod
echo "是否更新pod，更新将花费一些时间(最好每次打包都更新pod文件)？"

select update in "Y" "N"
do
case ${update} in
"Y")
echo "\033[37m pod文件更新... \033[0m"
pod update
pod install
echo "\033[37m pod文件更新完毕 \033[0m"
break
;;
*)
echo "\033[37m 不更新pod文件 \033[0m"
break
;;
esac
break
done;

echo


echo "################打包基本信息###################"
#时间戳
timeStamp="$(date +"%Y%m%d%H%M%S")"
echo "\033[37m 当前时间:"$(date)" \033[0m"

#设置display name
/usr/libexec/PlistBuddy -c "Set :CFBundleDisplayName ${display_name}" "${app_infoplist_path}"
echo "\033[37m app显示名:"${display_name}" \033[0m"

#设置bundle short version
/usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString ${VERSION}" "${app_infoplist_path}"
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" "${app_infoplist_path}")
echo "\033[37m 版本号:"${bundleShortVersion}" \033[0m"

#设置bundle version
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion ${timeStamp}" "${app_infoplist_path}"
bundleVersion=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${app_infoplist_path}")
echo "\033[37m 编译版本:"${bundleVersion}" \033[0m"


#打包配置(Release|Debug)
build_configuration=Release
build_scheme=${project_name}

if [ ${type} == ${DEV} ]
then
    build_scheme=${project_name}_DEV
    provisioningProfile="**"#这里根据自己项目修改
    echo "\033[37m 编译环境:DEV \033[0m"
elif [ ${type} == ${UAT} ]
then
    provisioningProfile="**"#这里根据自己项目修改
    echo "\033[37m 编译环境:UAT \033[0m"
elif [ ${type} == ${APPSTORE} ]
then
    provisioningProfile="**"#这里根据自己项目修改
    echo "\033[37m 编译环境:APPSTORE \033[0m"
fi

echo
echo "\033[37m 以上为本次打包的app基本信息，是否确认？(Y(确认打包)/N(取消退出打包)) \033[0m"
select yes in "Y" "N"
do
case ${yes} in
"Y")
echo "确认打包"
break
;;
"N")
echo "取消退出打包"
svn revert -R ${app_infoplist_path}
exit
;;
esac
done

#打包开始
echo "################开始打包，大概将会编译几分钟时间，请耐心等待...###################"

function image_revert() {
    image_name=$1

    target_path=$(find ${project_path}/${project_name} -name ${image_name})

    svn revert -R ${target_path}
}

#图片恢复
image_revert "AppIcon.appiconset"

#图片转换
if [ ${type} == ${DEV} ]
then

    temp_convert_image_dir=${project_path}/temp_convert_image_dir
    rm -rf ${temp_convert_image_dir}
    mkdir ${temp_convert_image_dir}

    function image_convert() {
        image_name=$1

        target_path=$(find ${project_path}/${project_name} -name ${image_name})

        cp ${target_path} ${temp_convert_image_dir}

        convert_path=${temp_convert_image_dir}/${image_name}
        convert ${convert_path} -fill white -font Times-Bold -pointsize 40 -gravity south -annotate 0 "${bundleShortVersion}" ${convert_path}


        cp ${convert_path} ${target_path}
    }

    #转换图片
    image_convert "AppIcon29x29@2x.png"
    image_convert "AppIcon29x29@3x.png"
    image_convert "AppIcon40x40@2x.png"
    image_convert "AppIcon40x40@3x.png"
    image_convert "AppIcon60x60@2x.png"
    image_convert "AppIcon60x60@3x.png"

    rm -rf ${temp_convert_image_dir}
fi

startDate=$(date +%Y%m%d%H%M%S)

#build clean
echo "\033[37m build clean... \033[0m"
build_clean="xcodebuild clean -workspace ${build_workspace} -scheme ${build_scheme} -configuration ${build_configuration}"
echo ${build_clean}
${build_clean} > /dev/null
echo "\033[37m build clean finished \033[0m"

#build archive
echo "\033[37m build archive... \033[0m"
archive_name=${build_scheme}_${bundleShortVersion}_${bundleVersion}.xcarchive
archive_path=${project_path}/Archive/${archive_name}

build_workspace="xcodebuild -workspace ${build_workspace} -scheme ${build_scheme} -configuration ${build_configuration} -destination generic/platform=iOS archive ONLY_ACTIVE_ARCH=NO -archivePath ${archive_path} ${GCC_PREPROCESSOR_DEFINITIONS}"
echo ${build_workspace}
${build_workspace} > /dev/null

echo "################${archive_name}生成完成，请保留此文件，以后查找BUG使用!!!!###################"
echo

#图片转换
if [ ${type} == ${DEV} ]
then

    image_revert "AppIcon.appiconset"

fi

echo
echo "################开始导出ipa文件...###################"

#export ipa

if [ ${type} == ${DEV} ]
then
    ipa_path=${project_path}/DEV
    type_name="**"#这里根据自己项目修改
elif [ ${type} == ${UAT} ]
then
    ipa_path=${project_path}/UAT
    type_name="**"#这里根据自己项目修改
elif [ ${type} == ${APPSTORE} ]
then
    ipa_path=${project_path}/APPSTORE
    type_name="**"#这里根据自己项目修改
fi
rm -rf ${ipa_path}
mkdir ${ipa_path}

ipa_name=${build_scheme}_${bundleShortVersion}_${bundleVersion}.ipa
ipa_path=${ipa_path}/${ipa_name}

echo "\033[37m ipa_name:"${ipa_name}" \033[0m"
echo

build_ipa="xcodebuild -exportArchive -exportFormat ipa -archivePath ${archive_path} -exportPath ${ipa_path} -exportProvisioningProfile ${provisioningProfile} "
echo ${build_ipa}

#log path
log_path=${project_path}/Log
rm -rf ${log_path}
mkdir ${log_path}

build_ipa_log=${log_path}/build_ipa.log
rm -rf ${project_path}/Log
mkdir ${project_path}/Log

${build_ipa} > ${build_ipa_log}

if [ ! -f "${ipa_path}" ]
then
    echo "################打包失败 查看错误结果:${ipa_log}###################"
exit
else
    rm -rf ${log_path}
    echo "\033[37m 导出ipa文件完成 所在地址:${ipa_path} \033[0m"
    echo "################导出ipa文件完成 所在地址:${ipa_path}###################"
fi

echo
echo
echo
endDate=$(date +%Y%m%d%H%M%S)
let date=${endDate}-${startDate}
echo "################打包共用时:"${date}"秒################"


#svn commit
#这里根据自己项目修改，我们使用的是svn
echo "是否提交打包版本信息(正常打包应该都选择Y)？(Y(提交)/N(跳过不提交))"
select commit in "Y" "N"
do
case ${commit} in
"Y")
echo "提交本地版本修改..."
svn ci ${app_infoplist_path} -m "自动打包上传version:${bundleShortVersion} buildversion:${bundleVersion} 类型:${type_name} 内容:${content}"
echo "提交完毕"
break
;;
*)
echo "未提交本地修改"
break
;;
esac
done

#上传APPSTORE平台
#成功后返回的内容包括<key>success-message</key>，失败其他消息

altool_path=/Applications/Xcode.app/Contents/Applications/Application\ Loader.app/Contents/Frameworks/ITunesSoftwareService.framework/Versions/A/Support/altool

function validateApp () {
    echo "验证ipa正确性..."
    validate=$("${altool_path}" --validate-app -f ${ipa_path} -u ${userName} -p ${password} --output-format xml > out)
    echo "validateApp:${validate}"
    ${validate}

    message=$(cat out | grep "<key>success-message</key>")
    if [ "${message}" == "" ]
    then
        echo "验证失败，具体失败内容如下:$(cat out)"
        rm out
        echo "重新验证、不管错误跳过验证还是退出?(Y(重新验证)/J(跳过验证)/Exit(退出))"

        select validate in "Y" "J" "Exit"
        do
        case ${validate} in
        "Y")
        echo "重新验证"
        validateApp
        break
        ;;
        "J")
        echo "跳过验证"
        break
        ;;
        "Exit")
        echo "退出"
        exit
        ;;
        esac
        done
    else
       rm out
       echo "验证完毕"
    fi

}

function uploadApp () {
    upload=$("${altool_path}" --upload-app -f ${ipa_path} -u ${userName} -p ${password} --output-format xml > out)
    echo "uploadApp:${upload}"
    ${upload}

    message=$(cat out | grep "<key>success-message</key>")

    if [ "${message}" == "" ]
    then
        echo "上传失败，具体失败内容如下:$(cat out)"
        rm out
        echo "重新上传或者退出?(Y(重新验证)/Exit(退出))"

        select validate in "Y" "Exit"
        do
        case ${validate} in
        "Y")
        echo "开始重新上传"
        validateApp
        break
        ;;
        "Exit")
        echo "上传完毕"
        exit
        ;;
        esac
        done
    else
        rm out
        echo "上传成功"
    fi
}

echo
echo

if [ ${type} == ${APPSTORE} ]
then
echo "################发布到APPSTORE###################"
if [ ! -d "/usr/local/itms" ]
then
echo "创建超链接"
ln -s /Applications/Xcode.app/Contents/Applications/Application\ Loader.app/Contents/itms /usr/local/itms
echo "超链接创建成功"
fi

#验证ipa
validateApp

#上传ipa文件
echo "上传appStore..."
uploadApp
exit
fi

#上传蒲公英平台
echo
echo
echo "################发布到蒲公英分发平台###################"
#蒲公英设置
#这里根据自己项目修改，我们使用的是蒲公英
echo "################蒲公英key设置###################"
uKey="**"#这里根据自己项目修改
api_key="**"#这里根据自己项目修改
echo "\033[37m uKey:"${uKey}" \033[0m"
echo "\033[37m api_key:"${api_key}" \033[0m"
echo "\033[37m updateDescription:"${content}" \033[0m"
echo
echo "是否上传到蒲公英分发平台上?"
select confirm in "Y" "N"
do
case ${confirm} in
"Y")
echo "\033[37m 正在上传... \033[0m"

curl -F "file=@${ipa_path}" -F "updateDescription=${content}" -F "uKey=${uKey}" -F "_api_key=${api_key}" http://www.pgyer.com/apiv1/app/upload
echo
echo "\033[37m 上传完成,请至蒲公英平台查看 \033[0m"
echo "请将下面信息发布到qq群中，提醒大家进行更新安装,首先自己测试下下载地址是否可用"

if [ ${type} == ${DEV} ]
then
app_url="**"#这里根据自己项目修改
elif [ ${type} == ${UAT} ]
then
app_url="**"#这里根据自己项目修改
fi

echo "IOS ${type_name}版本:${VERSION} buildversion:${bundleVersion} 下载地址：http://www.pgyer.com/${app_url} 更新内容见网页"
exit
;;
*)
echo "################运行完毕###################"
exit
;;
esac
done




