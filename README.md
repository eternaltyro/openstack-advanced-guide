/*
# Copyright [2013] [Kiran, Johnson, Atul, Suseendran, Yogesh]
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
*/

openstack-advanced-guide
========================

OpenStack advanced guide covering, high availability, security, multi-node, seamless addition of components, fault tolerance, replication, etc.

Contents
--------
* Essex Guide imported from Launchpad 
* Grizzly Guide
* Advanced OpenStack configuration [coming soon]

Making the PDF:
---------------

Install dblatex (700 MB) on Ubuntu

    $ sudo apt-get -y install dblatex

Compiling a single xml docbook file into PDF:

    $ dblatex filename.xml

Compiling the entire book: Simply compile the index file

    $ dblatex Openstackbook.xml

Lighter options:
 
    $ sudo apt-get -y install docbook docbook-xsl-ns xsltproc fop xmlto libxml2-utils xmlstarlet
    $ xmlstarlet val --err --xsd /usr/share/xml/docbook/schema/xsd/5.0/docbook.xsd book.xml
    $ xsltproc /usr/share/xml/docbook/stylesheet/docbook-xsl-ns/fo/docbook.xsl filename.xml > filename.fo
    $ fop -fo book.fo -pdf book.pdf

Alternatively, you can use the web-based docbook to PDF conversion tool at http://docbookpublishing.com/

