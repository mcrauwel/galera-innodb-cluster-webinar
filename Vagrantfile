# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))
SUBNET = ENV['SUBNET']

ROOT_PASSWORD='MySup3rS3cr3tP4ssw0rd!!'
APP_PASSWORD='4PPp4ssw0rd!!'
MIGRATION_PASSWORD='M1gr4t10n!!'
REPLICATION_PASSWORD='R3pl1c4t10n!!'

MARIADB_MYSQL_REPLICATION_MASTER='galera02'

Vagrant.configure("2") do |config|

  (1..3).each do |i|
    config.vm.define "galera0#{i}" do |node|
      node.vm.box = "centos/7"
      node.vm.hostname = "galera0#{i}"
      node.vm.network "private_network", ip: "172.16.#{SUBNET}.6#{i}"

      # node.vm.provider "virtualbox" do |vb|
      #   vb.memory = "1024"
      # end
      #
      node.vm.provision "shell", inline: <<-SHELL
        echo "hello from node #{i}"

        echo "127.0.0.1 localhost" > /etc/hosts
        echo "172.16.#{SUBNET}.51 app01" >> /etc/hosts
        echo "::1 localhost" >> /etc/hosts
        for i in {1..3}
        do
          echo "172.16.#{SUBNET}.6$i galera0$i" >> /etc/hosts
        done
        for i in {1..3}
        do
          echo "172.16.#{SUBNET}.7$i innodbcluster0$i" >> /etc/hosts
        done

        curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup |
          sudo bash -s -- --mariadb-server-version="mariadb-10.3" --mariadb-maxscale-version="2.3"

        sudo yum install -y --quiet MariaDB-server MariaDB-client MariaDB-backup socat

        sed -i s/^SELINUX=.*$/SELINUX=permissive/ /etc/selinux/config
        setenforce permissive

        if [[ ! -f /var/lib/mysql/grastate.dat ]]
        then

          if [[ -f /etc/my.cnf.d/wsrep.cnf ]]
          then
            rm /etc/my.cnf.d/wsrep.cnf
          fi

          systemctl enable mysql
          systemctl start mysql

          if [[ #{i} -eq 1 ]]
          then
            mysql --execute="CREATE USER IF NOT EXISTS sstuser@localhost IDENTIFIED BY 'passw0rd'"
            mysql --execute="GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';"
          fi

          sed 's/\#\{SUBNET\}/#{SUBNET}/g' /vagrant/wsrep.cnf > /etc/my.cnf.d/wsrep.cnf
          sed -i 's/\#\{i\}/#{i}/g' /etc/my.cnf.d/wsrep.cnf
          sed 's/\#\{i\}/#{i}/g' /vagrant/binary-logging.cnf > /etc/my.cnf.d/binlog.cnf

          echo "" >> /etc/my.cnf.d/server.cnf
          echo "[mysqld]" >> /etc/my.cnf.d/server.cnf
          echo "skip-name-resolve" >> /etc/my.cnf.d/server.cnf
          echo "character-set-server = utf8mb4" >> /etc/my.cnf.d/server.cnf

          if [[ #{i} -eq 1 ]]
          then
            systemctl stop mysql && galera_new_cluster
          else
            systemctl restart mysql
          fi

        fi

        if [[ #{i} -eq 1 ]]
        then
          TMPFILE=`mktemp`

          # drop default users
          echo "DROP USER IF EXISTS root@galera0#{i};" >> ${TMPFILE}
          echo "DROP USER IF EXISTS ''@galera0#{i};" >> ${TMPFILE}
          echo "DROP USER IF EXISTS ''@localhost;" >> ${TMPFILE}

          # create root user
          echo "ALTER USER root@'localhost' IDENTIFIED BY '#{ROOT_PASSWORD}';" >> ${TMPFILE}
          echo "ALTER USER root@'127.0.0.1' IDENTIFIED BY '#{ROOT_PASSWORD}';" >> ${TMPFILE}
          echo "ALTER USER root@'::1' IDENTIFIED BY '#{ROOT_PASSWORD}';" >> ${TMPFILE}
          echo "CREATE USER IF NOT EXISTS root@'172.16.#{SUBNET}.0/255.255.255.0' IDENTIFIED BY '#{ROOT_PASSWORD}';" >> ${TMPFILE}
          echo "GRANT ALL ON *.* TO root@'172.16.#{SUBNET}.0/255.255.255.0' WITH GRANT OPTION;" >> ${TMPFILE}


          # app user and grants
          echo "CREATE DATABASE IF NOT EXISTS sbtest;" >> ${TMPFILE}
          echo "CREATE USER IF NOT EXISTS sbtest@'172.16.#{SUBNET}.0/255.255.255.0' IDENTIFIED BY '#{APP_PASSWORD}';" >> ${TMPFILE}
          echo "GRANT ALL ON sbtest.* TO sbtest@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}

          # maxscale grants
          echo "CREATE USER IF NOT EXISTS maxscale@'172.16.#{SUBNET}.0/255.255.255.0' IDENTIFIED BY 'M4xsc4l3p4ss!!';" >> ${TMPFILE}
          echo "GRANT SELECT ON mysql.user TO maxscale@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}
          echo "GRANT SELECT ON mysql.roles_mapping TO maxscale@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}
          echo "GRANT SELECT ON mysql.db TO maxscale@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}
          echo "GRANT SELECT ON mysql.tables_priv TO maxscale@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}
          echo "GRANT SHOW DATABASES ON *.* TO maxscale@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}


          # create migration user
          echo "CREATE USER IF NOT EXISTS migration@localhost IDENTIFIED BY '#{MIGRATION_PASSWORD}';" >> ${TMPFILE}
          echo "GRANT ALL ON *.* TO migration@localhost;" >> ${TMPFILE}
          echo "CREATE USER IF NOT EXISTS replication@'172.16.#{SUBNET}.0/255.255.255.0' IDENTIFIED BY '#{REPLICATION_PASSWORD}';" >> ${TMPFILE}
          echo "GRANT REPLICATION SLAVE ON *.* TO replication@'172.16.#{SUBNET}.0/255.255.255.0';" >> ${TMPFILE}

          cat ${TMPFILE} | mysql
        fi

        echo "[client]
user = root
password = #{ROOT_PASSWORD}
" > /root/.my.cnf

        mkdir -p /root/.ssh

        cp /vagrant/id_rsa /root/.ssh/id_rsa
        cp /vagrant/id_rsa.pub /root/.ssh/id_rsa.pub

        chown root:root /root/.ssh/id_rsa*
        chmod 400 /root/.ssh/id_rsa

        cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys

      SHELL
    end
  end

  (1..3).each do |i|
    config.vm.define "innodbcluster0#{i}" do |node|
      node.vm.box = "centos/7"
      node.vm.hostname = "innodbcluster0#{i}"
      node.vm.network "private_network", ip: "172.16.#{SUBNET}.7#{i}"


      node.vm.provision "shell", inline: <<-SHELL
        echo "hello from node #{i}"

        echo "127.0.0.1 localhost" > /etc/hosts
        echo "::1 localhost" >> /etc/hosts
        echo "172.16.#{SUBNET}.51 app01" >> /etc/hosts
        for i in {1..3}
        do
          echo "172.16.#{SUBNET}.6$i galera0$i" >> /etc/hosts
        done
        for i in {1..3}
        do
          echo "172.16.#{SUBNET}.7$i innodbcluster0$i" >> /etc/hosts
        done


        sudo yum install -y --quiet https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
        sudo yum install -y --quiet mysql-community-server mysql-shell

        sed -i s/^SELINUX=.*$/SELINUX=permissive/ /etc/selinux/config
        setenforce permissive

        echo "server-id = 1#{i}" >> /etc/my.cnf

        systemctl enable mysqld
        systemctl start mysqld

        if [[ ! -f /root/.my.cnf ]]
        then
          TEMP_PASS=`cat /var/log/mysqld.log | grep "temporary pass" | awk '{ print $NF }'`
          mysql -uroot -p${TEMP_PASS} --connect-expired-password --execute="ALTER USER root@localhost IDENTIFIED WITH mysql_native_password BY '#{ROOT_PASSWORD}'"
        fi

        echo "[client]
user = root
password = #{ROOT_PASSWORD}
" > /root/.my.cnf

        for i in {1..3}
        do
          mysql --execute="CREATE USER IF NOT EXISTS root@172.16.#{SUBNET}.7${i} IDENTIFIED WITH mysql_native_password BY '#{ROOT_PASSWORD}'"
          mysql --execute="GRANT ALL PRIVILEGES ON *.* TO 'root'@'172.16.#{SUBNET}.7${i}' WITH GRANT OPTION"
        done

        mkdir -p /root/.ssh

        cp /vagrant/id_rsa /root/.ssh/id_rsa
        cp /vagrant/id_rsa.pub /root/.ssh/id_rsa.pub

        chown root:root /root/.ssh/id_rsa*
        chmod 400 /root/.ssh/id_rsa

        cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys

        if [[ #{i} -eq 1 ]]
        then
          ssh -o StrictHostKeyChecking=no #{MARIADB_MYSQL_REPLICATION_MASTER} "mysqldump -umigration -p'#{MIGRATION_PASSWORD}' --single-transaction --master-data sbtest" | mysql --init-command="CREATE DATABASE IF NOT EXISTS sbtest; USE sbtest; CHANGE MASTER TO master_host = '#{MARIADB_MYSQL_REPLICATION_MASTER}', master_user = 'replication', master_password = '#{REPLICATION_PASSWORD}';"
          mysql --execute="CHANGE REPLICATION FILTER REPLICATE_WILD_DO_TABLE = ('sbtest.%');"
          mysql --execute="START SLAVE;"
        fi

      SHELL
    end
  end

  config.vm.define "app01" do |node|
    node.vm.box = "centos/7"
    node.vm.hostname = "app01"
    node.vm.network "private_network", ip: "172.16.#{SUBNET}.51"

    node.vm.provision "shell", inline: <<-SHELL
      echo "hello from app01"

      echo "127.0.0.1 localhost" > /etc/hosts
      echo "::1 localhost" >> /etc/hosts
      echo "172.16.#{SUBNET}.51 app01" >> /etc/hosts
      for i in {1..3}
      do
        echo "172.16.#{SUBNET}.6$i galera0$i" >> /etc/hosts
      done
      for i in {1..3}
      do
        echo "172.16.#{SUBNET}.7$i innodbcluster0$i" >> /etc/hosts
      done

      curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup |
        sudo bash -s -- --mariadb-server-version="mariadb-10.3" --mariadb-maxscale-version="2.3"

      sudo yum install -y --quiet https://repo.percona.com/yum/percona-release-latest.noarch.rpm
      sudo yum install -y --quiet https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
      sudo yum install -y --quiet sysbench mysql maxscale mysql-router-community net-tools


      sed -i s/^SELINUX=.*$/SELINUX=permissive/ /etc/selinux/config
      setenforce permissive

      sed 's/\#\{SUBNET\}/#{SUBNET}/g' /vagrant/maxscale.cnf > /etc/maxscale.cnf
      systemctl enable maxscale
      systemctl start maxscale


      mysql -h galera01 -P3306 -uroot -p'#{ROOT_PASSWORD}' --execute='DROP DATABASE IF EXISTS sbtest; CREATE DATABASE IF NOT EXISTS sbtest;'

      sysbench oltp_read_write --mysql-host=app01 --mysql-port=3306 --mysql-password='#{APP_PASSWORD}' --tables=8 prepare

      echo "*/5 * * * * root /usr/bin/sysbench oltp_read_write --mysql-host=app01 --mysql-port=3306 --mysql-password='#{APP_PASSWORD}' --tables=8 --time=295 --report-interval=3 --threads=4 run > /var/log/app.log 2>&1" > /etc/cron.d/app

      cp /vagrant/id_rsa /root/id_rsa
      cp /vagrant/id_rsa.pub /root/id_rsa.pub

      chown root:root /root/id_rsa*
      chmod 400 /root/id_rsa

      mkdir -p /root/.ssh
      cat /root/id_rsa.pub > /root/.ssh/authorized_keys

    SHELL
  end
end
