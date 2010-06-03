=============================================================
Installing Multicore Solr 1.3.0 / Tomcat 6.0.X on Ubuntu 8.10
=============================================================

Install OpenJDK (comes from universe)::

    aptitude install openjdk-6-jre

Download Tomcat 6.X and move it in place::

    wget http://apache.siamwebhosting.com/tomcat/tomcat-6/v6.0.20/bin/apache-tomcat-6.0.20.tar.gz
    tar zxvf apache-tomcat-6.0.20.tar.gz
    sudo mv apache-tomcat-6.0.20 /usr/local/tomcat-6.0.20

Set up an init.d script (/etc/init.d/tomcat)::

    # Tomcat auto-start
    #
    # description: Auto-starts tomcat
    # processname: tomcat
    # pidfile: /var/run/tomcat.pid

    TOMCAT_ROOT=/usr/local/tomcat-6.0.20
    export JAVA_HOME=/usr/lib/jvm/java-6-openjdk/

    case $1 in
    start)
        sh $TOMCAT_ROOT/bin/startup.sh
        ;;
    stop)
        sh $TOMCAT_ROOT/bin/shutdown.sh
        ;;
    restart)
        sh $TOMCAT_ROOT/bin/shutdown.sh
        sh $TOMCAT_ROOT/bin/startup.sh
        ;;
    esac
    exit 0

Set permissions::

    sudo chmod 755 /etc/init.d/tomcat

Hook it up to start-up::

    sudo ln -s /etc/init.d/tomcat /etc/rc1.d/K99tomcat
    sudo ln -s /etc/init.d/tomcat /etc/rc2.d/S99tomcat

Run Tomcat::

    sudo /etc/init.d/tomcat start

Download / install Solr 1.3.0::

    wget http://apache.siamwebhosting.com/lucene/solr/1.3.0/apache-solr-1.3.0.tgz
    tar zxvf apache-solr-1.3.0.tgz
    sudo cp apache-solr-1.3.0/example/webapps/solr.war /usr/local/tomcat-6.0.20/webapps/

Create solr.xml context in /usr/local/tomcat-6.0.20/conf/Catalina/localhost/::

    <Context docBase="/usr/local/tomcat-6.0.20/webapps/solr.war" debug="0" crossContext="true" >
       <Environment name="solr/home" type="java.lang.String" value="/var/solr/" override="true" />
    </Context>

Create the multicore stuff::

    sudo mkdir /var/solr
    sudo cp ~/apache-solr-1.3.0/example/multicore/solr.xml /var/solr/

Let's create a new core for a single site we want to host::

    sudo mkdir /var/solr/code.pinaxproject.com
    sudo cp -r ~/apache-solr-1.3.0/example/solr/conf /var/solr/code.pinaxproject.com/

Edit the /var/solr/solr.xml configuration file to contain something like::

    <cores adminPath="/admin/cores">
        <core name="code.pinaxproject.com" instanceDir="code.pinaxproject.com" />
    </cores>

Restart Tomcat::

    /etc/init.d/tomcat restart

Access things::

    http://yourserver:8080/solr

If you use haystack you'd set ``HAYSTACK_SOLR_URL`` to something like::

    HAYSTACK_SOLR_URL = "http://yourserver:8080/solr/code.pinaxproject.com"

Enjoy!