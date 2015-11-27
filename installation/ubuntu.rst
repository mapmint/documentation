.. MapMint Installation

Installation guide
===================================

Use the following instructions to install MapMint on Ubuntu 14.04 LTS
distribution.

.. toctree::
   :maxdepth: 3

All the instructions given in the following suppose that you are logged in
as a basic UNIX user, not root.

Initial settings
----------------

We will refer to the ``$SRC`` environment variable in all the steps
described in this documentation. So please, make sure to run the
following command in the same terminal until the end of the setup
process. In case you need another terminal, then please make sure to
redefine your ``$SRC`` environment variable by using the following
command.

.. code-block:: guess

    export SRC=/home/djay/src
    kdir $SRC

Download MapMint
-----------------------------

Use the following command to download the MapMint source code, you
will need some patch it contains to apply on the MapServer source code
later.

.. code-block:: guess
    
    cd $SRC/
    git clone https://github.com/mapmint/mapmint.git

Dependencies
------------

MapMint depends on various applications and libraries, some are
available as binary packages on Ubuntu, some require specific setup
procedure and some other require to be build by hand. In this section,
each step will be described.

Install binary packages
***********************

Use the following command to install the required binary packages.

.. code-block:: guess

    sudo apt-get install flex bison libfcgi-dev libxml2 libxml2-dev \
    curl openssl autoconf apache2 python-software-properties \
    subversion git libmozjs185-dev python-dev build-essential \
    libfreetype6-dev libproj-dev libgdal1-dev libcairo2-dev \
    apache2-dev libxslt1-dev python-cheetah cssmin python-psycopg2 \
    python-gdal python-libxslt1 postgresql-9.3  r-base cmake gdal-bin \
    libapache2-mod-fcgid ghostscript xvfb


Install R packages
******************

MapMint use specific R packages for giving access to the
discretisation options when styling a layer. To install the required
packages, use the following commands:

.. code-block:: guess

    sudo R
    install.packages("e1071")
    install.packages("classInt")
    q()

Install MapServer
*****************

`MapServer <http://mapserver.org>`__ is the cartographic engine used
in MapMint for publishing spatial data through standard OGC Web
Services. By now, MapMint depends on MapServer 6.2.0. Use the
following commands to download, build and install MapServer on your
platform.

.. code-block:: guess

    cd $SRC
    wget http://download.osgeo.org/mapserver/mapserver-6.2.0.tar.gz
    tar -xvf mapserver-6.2.0.tar.gz
    cd mapserver-6.2.0
    ./configure --with-wfs --with-python --with-freetype=/usr/ --with-ogr \
        --with-gdal --with-proj --with-geos --with-cairo --with-kml \
        --with-wmsclient --with-wfsclient --with-wcs --with-sos \
        --with-python=/usr/bin/python2.7 --without-gif --with-apache-module \
        --with-apxs=/usr/bin/apxs2 --with-apr-config=/usr/bin/apr-1-config \
	--enable-python-mapscript --with-zlib --prefix=/usr/



Install MapCache
****************

`MapCache <http://www.mapserver.org/en/mapcache/>`__ is a server used
in MapMint to provide tile caching service to speed up access to WMS
layers.

.. code-block:: guess

    cd $SRC
    git clone https://github.com/mapserver/mapcache.git
    cd mapcache/
    cmake .
    make 
    sudo make install
    
    wget http://geolabs.fr/dl/mapcache.xml
    sudo cp mapcache.xml /usr/lib/cgi-bin/mm/


Create or edit the ``/etc/ld.so.conf.d/zoo.conf`` and add a line
containing:``/usr/local/lib``, then run the following command:

.. code-block:: guess

    sudo ldconfig

Install MapMint components
--------------------------

MapMint is based on a suite of Web Services, despite most of them are
implemented using Python and JavaScript language there are some C
services which are required by the MapMint software. In this section,
the build procedure for the required services are presented.

Install ZOO-Project and the required C Services
***********************************************

`ZOO-Project <http://zoo-project.org>`__ is the WPS implementation
used for handling MapMint services.

The following ZOO-Services are required in the MapMint application:

  * ogr2ogr : vector conversion
  * base-vect-ops : basic vector operations (buffer, centroid ...)
  * gdal/dem : analyze and visualize DEMs (`ref
    <http://www.gdal.org/gdaldem.html>`__) 
  * gdal/grid : creates regular grid from the scattered data (`ref
    <http://www.gdal.org/gdal_grid.html>`__) 
  * gdal/contour: builds vector contour lines from a raster elevation
    model (`ref <http://www.gdal.org/gdal_contour.html>`__)
  * gdal/profile: compute elevation profile along a line
  * gdal/translate: converts raster data between different formats
    (`ref <http://www.gdal.org/gdal_translate.html>`__) 
  * gdal/warp: image reprojection and warping utility (`ref
    <http://www.gdal.org/gdalwarp.html>`__) 

.. code-block:: guess

    svn checkout http://www.zoo-project.org/svn/trunk zoo
    cd $SRC/zoo/thirds/cgic206
    sed "s:lib64:lib:g" -i Makefile 
    make
    cd $SRC/zoo/zoo-project/zoo-kernel
    autoconf
   ./configure --with-mapserver=$SRC/mapserver-6.2.0/ --with-python --with-pyvers=2.7 --with-js=/usr/ --with-xsltconfig=/usr/bin/xslt-config
    sed "s:/usr/lib/x86_64-linux-gnu/libapr-1.la::g" -i ZOOMakefile.opts
    make
    make install
    ldconfig
    cp zoo_loader.cgi ../../../mapmint/mapmint-services/
    
    cd $SRC/zoo/zoo-project/zoo-services/ogr/ogr2ogr
    make
    cp cgi-env/* $SRC/mapmint/mapmint-services/vector-converter/
    
    cd $SRC/zoo/zoo-project/zoo-services/ogr/base-vect-ops
    make
    cp cgi-env/* $SRC/mapmint/mapmint-services/vector-tools/
    
    cd $SRC/zoo/zoo-project/zoo-services/gdal
    for i in contour dem grid profile translate warp ; do
        echo $i
        cd $i
        make; sudo cp cgi-env/* $SRC/mapmint/mapmint-services/raster-tools/
        cd ..
    done

Install QREncode service
************************

The QREncode service require to be linked to a specific version of the qrencode library, it is the reason why it is necessary to dedicate a specific section for it.

.. code-block:: guess

    cd $SRC
    wget http://fukuchi.org/works/qrencode/qrencode-3.4.1.tar.gz
    tar xvf qrencode-3.4.1.tar.gz
    cd qrencode-3.4.1
    ./configure
    make 
    sudo make install
    sudo ldconfig -v
    cd $SRC/zoo/zoo-project/zoo-services/qrencode/
    make
    sudo cp cgi-env/* $SRC/mapmint/mapmint-services/


Build the MapMint C Services
****************************

Some MapMint services are implemented using the C language which means that you will need to build them. Use the following command to do so.

.. code-block:: guess

    cd $SRC/mapmint/mapmint-services
    for i in *-src ; do
        echo $i
        cd $i
        autoconf
        ./configure --with-zoo-kernel=$SRC/zoo/zoo-project/zoo-kernel --with-mapserver=$SRC/mapserver-6.2.0
        make
        cd ..
    done


Final setup
------------

Apache settings
***************

Since the begining everything was installed in $SRC/mapmint/. Nevertheless, the default Apache configuration suppose that you use a specific directory to store your HTML pages and your cgi scripts.

.. code-block:: guess

    sudo ln -s $SRC/mapmint/mapmint-ui/ /var/www/html/ui
    sudo ln -s $SRC/mapmint/public_map/ /var/www/html/pm
    
    sudo ln -s $SRC/mapmint/mapmint-services/ /usr/lib/cgi-bin/mm
    
    sudo a2enmod fcgid
    sudo a2enmod cgid
    sudo a2enmod rewrite


Edit the apache2 configuration file with your favorite text editor. Then, make sure the Directory block for ``/var/www`` looks like the following:

.. code-block:: guess

    <Directory /var/www/>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
    </Directory>
    
    FcgidInitialEnv "MAPCACHE_CONFIG_FILE" "/usr/lib/cgi-bin/mm/mapcache.xml"
    ScriptAlias /cache      /usr/lib/cgi-bin/mm/mapcache.fcgi
    
    <Location /cache>
            Order Allow,Deny
            Allow from all
            SetHandler fcgid-script
    </Location>

Edit the file ``/etc/apache2/conf-available/serve-cgi-bin.conf``,
then replace ``+SymLinksIfOwnerMatch`` by ``+FollowSymLinks``.

Restart the apache web server, by running the following command.

.. code-block:: guess

    sudo /etc/init.d/apache2 restart

Database setup
**************

Download and use databases required by MapMint, you are invited to
load the sql files related to the PostGIS module before loading the
mmdb.sql file (the location will depend on your setup).

.. code-block:: guess

    sudo mkdir /var/data
    cd /var/data
    sudo wget http://geolabs.fr/dl/mm.db
    
    sudo wget http://geolabs.fr/dl/mmdb.sql
    
    sudo /etc/init.d/postgresql start
    
    createdb -E utf-8 mmdb
    psql mmdb -f mmdb.sql

Directories settings
********************

MapMint requires some specific directories to be available, use the following commands to create them.

.. code-block:: guess

    sudo cp -r $SRC/mapmint/template/data/* /var/data
    sudo mkdir /var/data/{templates,dirs,public_maps,georeferencer_maps}
    sudo mkdir -p /var/www/html/tmp/descriptions
    sudo mkdir -p /var/www/html/pm/styles
    sudo mkdir -p /var/cache/zoo-project
    
    cd $SRC
    wget http://geolabs.fr/dl/fonts.tar.bz2
    sudo mkdir /var/data/fonts
    sudo tar -xvf fonts.tar.bz2 -C /var/data/fonts
    
    cp $SRC/mapmint/mapmint-ui/js/.htaccess $SRC/mapmint/public_map/
    sudo chown -R www-data:www-data /var/data
    sudo chown www-data:www-data /usr/lib/cgi-bin/mm/main.cfg
    sudo chown www-data:www-data /usr/lib/cgi-bin/mm/mapcache.xml
    sudo chown -R www-data:www-data /var/www/html/pm/styles


MapMint initial settings
************************

MapMint runs on ZOO-Project so its main configuration file is the ``main.cfg`` file decribed `here <http://www.zoo-project.org/docs/kernel/configuration.html>`__. The most important sections are described bellow.

The [main] Section
..................

 * ``mmAddress``: the url to use to access your MapMint setup (``/ui/``)
 * ``dataPath``: the directory to store maps, fonts and datasources (``/var/data``)
 * ``tmpPath``: the directory to store temporary files (``/var/www/hml/tmp``)
 * ``tmpUrl``: the url to access the temporary files (``/tmp/``)
 * ``cacheDir``: the directory to cache files (``/var/cache/zoo-project``)
 * ``rootUrl``: the public url (``/ui/public/``)
 * ``templatesPath``: the full path to access the ``mapmint-ui/templates``
   directory (``/var/www/html/ui/templates/``)
 * ``publicationUrl``: the url to access your ``public_maps`` directory from
   your webserver (``/pm/``)
 * ``publicationPath``: the full path of the ``public_maps`` directory (``/var/www/html/pm/``)
 * ``mmPath``: the full path to the ``mapmint-ui`` directory on the server (``/var/www/html/ui/``)
 * ``sessPath``: the full path to store session files (``/tmp``)
 * ``applicationAddress``: the full url to access the ``mapmint-ui`` directory
   from the web server (``http://<YourHost>/ui/``)
 * ``templatesAddress``: url to access the geenrated templates for
   displaying informations about features (``http://<YouHost>/tmpl/``)
 * ``serverAddress``: the url to access the ZOO-Kernel (``http://<YouHost>/cgi-bin/mm/zoo_loader.cgi``)
 * ``mapserverAddress``: the url to access the MapServer (``http://<YouHost>/cgi-bin/mm/mapserv.cgi``)
 * ``Rpy2``: can take the value true or false depending on the availability of the Rpy2 module on the server (``true``)
 * ``encoding``: should be ``utf-8`` 
 * ``language``: the default language to use
 * ``lang``: available languages list (languages are separated by comma)
 * ``cookiePrefix``: the name of the cookie used by mapmint (``MMID``)
 * ``msOgcVersion``: version of the OGC services published through
   MapServer (WMS/WFS/WCS) (1.0.0)
 * ``dbuserName``: the name of an existing PostGIS datastore to access the
   user database (you will have to set it later, once you will create it from your adminitration interface)
 * ``dblink``: the full path to the sqlite database (``/var/data/mm.db)``)
 * ``dbuser``: can be the name of an available section in the ```main.cfg```
   defining the database connection parameters or dblink in case only
   sqlite is used (``mmdb``)
 * ``isTrial``: should take the value ``true`` (obsolete)
 * ``jsCache``: can take the value ``prod`` or ``dev`` and will define if the
   JavaScript files should be compressed on the fly or not (obsolete in 2.0)
 * ``cssCache``: can take the value ``prod`` or ``dev`` and will define if the
  CSS files should be compressed on the fly or not (obsolete in 2.0)
 * ``3D``: can take the value ``true`` or ``false`` to define if 3D capabilities
   should be activated or not (obsolete)


The database connection
.......................

This section is defined by administrator in the main.cfg (``mdb``) and refered
in dbuser as mentionned in the previous section. It should contains:

 * ``host``: the hostname / ip address / socket to connect the database (``/var/run/postgresql/``)
 * ``schema``: the schema used to store the mapmint tables (``mm``)
 * ``port``: the port number to connect the database (``5432``)
 * ``dbname``: the database name (``mmdb``)
 * ``user``: the user to connect the database (``postgres``)
 * ``password``: the password to connect the database (you have to define  it)


Install and start LibreOffice as a server
*****************************************

`LibreOffice <http://libreoffice.org>`__ is the application used in MapMint to generate documents. It runs as a server and some specific MapMint services are able to connect.

To install LibreOffice, download its binary version from one mirror
listed `here
<http://download.documentfoundation.org/libreoffice/stable/5.0.3/deb/x86_64/LibreOffice_5.0.3_Linux_x86-64_deb.tar.gz.mirrorlist>`__
and move the downloaded file in your ``$SRC``  directory. The run the
following command to setup the packages.

.. code-block:: guess

    tar -xvf LibreOffice_5.0.3_Linux_x86-64_deb.tar.gz
    cd LibreOffice*/DEBS/ 
    sudo dpkg -i *.deb

First start a screen named ``PaperMint``:

.. code-block:: guess

    screen -r -R PaperMint


From this screen, start a X Server and the LibreOffice server by running the following commands:

.. code-block:: guess

    Xvfb :11&
    export DISPLAY=:11
    soffice --nofirststartwizard --norestore --nocrashreport --headless "--accept=socket,host=127.0.0.1,port=3662;urp"


Now you can check in another terminal if the server is available,
press ``ctrl+a`` then ``c`` from  your screen to open a new
terminal. Then run the following command:

.. code-block:: guess

    netstat -na | grep 3662


You should see a line specifying that the 3662 port is on listen mode.

Then press ``ctrl+a`` then ``d`` to quit the screen but keeping it
available for future use.

Acess your MapMint instance
------------------------------

To access your brand new MapMint installation, use the following url:
http://[YourHost]/ui/Dashboard_bs and replace the [YourHost] part with
your local IP address or host name.

Initially, your admin login is ``test`` and the password is
``demo02``. You are invited to remove this default parameters from
the admin user interface once you log in the administration interface.





