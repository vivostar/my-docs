# 如何使用bigtop打包hudi的rpm包

环境： rockylinux 8.7
## 安装必备的软件

```bash
yum install -y git yum-utils ruby java-1.8.0-openjdk-devel zip unzip rpm-build

# 使用sdkman管理开发时使用的sdk工具
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk version
sdk install maven
```

## 下载bigtop代码

```bash
git clone https://github.com/apache/bigtop.git
```

## 修改bigtop.bom文件

修改bigtop的bigtop.bom文件，添加hudi的基本信息

```json
    'hudi' {
      name    = 'hudi'
      pkg     = name
      rpm_pkg_suffix = "_" + bigtop.base_version.replace(".", "_")
      version {
        base  = '0.13.0'
        pkg   = base
        release = 1
      }
      tarball {
        source      = "hudi-${version.base}.src.tgz"
        destination = "hudi-${version.base}-src.tar.gz"
      }
      url {
        download_path = "/hudi/${version.base}"
        site          = "${apache.APACHE_MIRROR}/${download_path}"
        archive       = "${apache.APACHE_ARCHIVE}/${download_path}"
      }
    }
```

## 添加hudi构建脚本

hudi的构建脚本如下所示
```bash
mvn clean package -DskipTests
```

在bigtop-packages/src/common目录下新增hudi目录，新增do-component-build
```bash
cd bigtop-packages/src/common
mkdir hudi
cd hudi
cat > ./do-component-build << __EOT__
set -xe

. `dirname $0`/bigtop.bom

mvn clean package -DskipTests
__EOT__
```

## 新增hudi的spec文件

```bash
%define hudi_name hudi
%define hudi_pkg_name hudi%{pkg_name_suffix}
%define hadoop_pkg_name hadoop%{pkg_name_suffix}
%define __os_install_post %{nil}

%define usr_lib_hudi %{parent_dir}/usr/lib/%{hudi_name}
%define etc_hudi %{parent_dir}/etc/%{hudi_name}

Name: %{hudi_pkg_name}
Version: %{hudi_version}
Release: %{hudi_release}
Summary:Apache Hudi is the next generation streaming data lake platform.
URL: http://hudi.apache.org
Group: Development/Libraries
Buildroot: %{_topdir}/INSTALL/%{name}-%{version}
License: Apache License v2.0
Source0: %{hudi_name}-%{hudi_base_version}-src.tar.gz
Source1: do-component-build
#BIGTOP_PATCH_FILES
BuildArch: noarch
Requires: %{hadoop_pkg_name} %{hadoop_pkg_name}-hdfs %{hadoop_pkg_name}-yarn %{hadoop_pkg_name}-mapreduce

%description
Apache Hudi is the next generation streaming data lake platform. 
Apache Hudi brings core warehouse and database functionality directly 
to a data lake. Hudi provides tables, transactions, efficient upserts and deletes,
advanced indexes, streaming ingestion services, data clustering/compaction 
optimizations, and concurrency all while keeping your data in open source file 
formats.

%define package_macro() \
%package %1 \
Summary: apache hudi %1 bundle \
Group: System/Daemons \
%description %1 \
apache hudi %1 bundle 

%package_macro spark
%package_macro flink
%package_macro datahub-sync
%package_macro timeline-server
%package_macro kafka-connect
%package_macro trino
%package_macro utilities
%package_macro utilities-slim
%package_macro aws
%package_macro gcp
%package_macro hadoop-mr
%package_macro presto
%package_macro cli
%package_macro hive-sync

%prep
%setup -q -n %{hudi_name}-%{hudi_base_version}

%build
bash %{SOURCE1}

%install
%__rm -rf $RPM_BUILD_ROOT

LIB_DIR=${LIB_DIR:-/usr/lib/hudi}
install -d -m 0755 $RPM_BUILD_ROOT/$LIB_DIR/lib

for comp in spark utilities utilities-slim cli        
do 
cp ./packaging/hudi-${comp}-bundle/target/hudi-${comp}-bundle_2.11-%{hudi_version}.jar $RPM_BUILD_ROOT/$LIB_DIR/lib
done

for comp in datahub-sync timeline-server kafka-connect trino gcp \
            aws hadoop-mr presto hive-sync          
do 
cp ./packaging/hudi-${comp}-bundle/target/hudi-${comp}-bundle-%{hudi_version}.jar $RPM_BUILD_ROOT/$LIB_DIR/lib
done

cp ./packaging/hudi-flink-bundle/target/hudi-flink1.16-bundle-%{hudi_version}.jar $RPM_BUILD_ROOT/$LIB_DIR/lib

%pre

# Manage configuration symlink
%post

%preun


#######################
#### FILES SECTION ####
#######################
# Scala Jar file management RPMs
%define jar_scala_macro() \
%files %1 \
%defattr(-,root,root) \
%{usr_lib_hudi}/lib/hudi-%1-bundle_2.11-%{hudi_version}.jar

# Java Jar file management RPMs
%define jar_macro() \
%files %1 \
%defattr(-,root,root) \
%{usr_lib_hudi}/lib/hudi-%1-bundle-%{hudi_version}.jar

%jar_scala_macro spark
%jar_macro datahub-sync
%jar_macro timeline-server
%jar_macro kafka-connect
%jar_macro trino
%jar_scala_macro utilities
%jar_scala_macro utilities-slim
%jar_macro aws
%jar_macro gcp
%jar_macro hadoop-mr
%jar_macro presto
%jar_scala_macro cli
%jar_macro hive-sync

%files flink
%defattr(-,root,root) 
%{usr_lib_hudi}/lib/hudi-flink1.16-bundle-%{hudi_version}.jar
```

## 构建hudi rpm包

上面步骤完成之后就可以进行构建

```bash
./gradlew allclean hudi-rpm
```
