Petrel
======

Tools for writing, submitting, debugging, and monitoring Storm topologies in pure Python.

Overview
========

Petrel offers some important improvements over the storm.py module provided with Storm:

* Topologies are implemented in 100% Python (using the generic JVMPetrel jar)
* Petrel's packaging support automatically sets up a Python virtual environment for your topology and makes it easy to install additional Python packages.
* "petrel.mock" allows testing of single components or single chains of related components.
* Petrel automatically sets up logging for every spout or bolt and logs a stack trace on unhandled errors.

Topology definition
===================

<pre>
import randomsentence
import splitsentence
import wordcount

def create(builder):
    builder.setSpout("spout", randomsentence.RandomSentenceSpout(), 1)
    builder.setBolt("split", splitsentence.SplitSentenceBolt(), 1).shuffleGrouping("spout")
    builder.setBolt("count", wordcount.WordCountBolt(), 1).fieldsGrouping("split", ["word"])
</pre>

Building and submitting topologies
==================================

Use the following command to package and submit a topology to Storm:

<pre>
petrel submit --sourcejar ../../jvmpetrel/target/storm-petrel-*-SNAPSHOT.jar --config localhost.yaml wordcount
</pre>

This command builds and submits a topology.

Build
-----

* Get the topology definition by loading the create.py script and calling create().
* Package a JAR containing the topology definition, code, and configuration.
* Files listed in manifest.txt, e.g. additional configuration files

Execute
-------

Petrel makes it easy to run multiple versions of a topology side by side, as long as the code differences are manageable by virtualenv. Before a spout or bolt starts up, Petrel creates a new Python virtualenv and runs the topology-specific setup.sh script to install Python packages. It is shared by all the spouts or bolts from that instance of the topology on that machine.

Monitoring
==========

Petrel provides a "status" command which lists the active topologies and tasks on a cluster. You can optionally filter by task name and Storm port (i.e. worker slot) number.

<pre>
petrel status 10.255.1.58
</pre>

Logging
=======

Petrel redirects stdout and stderr to the Python logger.

When Storm is running on a cluster, it can be nice to send certain messages (e.g. errors) to a central machine. Petrel provides the NIMBUS_HOST environment variable to help support this. For example, the following configuration declares a log handler which sends any worker log messages INFO or higher to the Nimbus host.

<pre>
[handler_hand02]
class=handlers.SysLogHandler
level=INFO
formatter=form02
args=((os.getenv('NIMBUS_HOST') or 'localhost',handlers.SYSLOG_UDP_PORT),handlers.SysLogHandler.LOG_USER)
</pre>

Petrel also has a "StormHandler" class sends messages to the Storm logger. This feature is currently not "released", but can be enabled by uncommenting the following line in petrel/util.py:

<pre>
#logging.StormHandler = StormHandler
</pre>


Testing
=======

Petrel provides a "mock" module which mocks some of Storm's features. This makes it possible to test individual components and simple topologies in pure Python, without relying on the Storm runtime.

<pre>
def test():
    bolt = WordCountBolt()
    
    from petrel import mock
    from randomsentence import RandomSentenceSpout
    mock_spout = mock.MockSpout(RandomSentenceSpout.declareOutputFields(), [
        ['word'],
        ['other'],
        ['word'],
    ])
    
    result = mock.run_simple_topology([mock_spout, bolt], result_type=mock.LIST)
    assert_equal(2, bolt._count['word'])
    assert_equal(1, bolt._count['other'])
    assert_equal([['word', 1], ['other', 1], ['word', 2]], result[bolt])
</pre>

In Petrel terms, a "simple" topology is one which only outputs to the default stream and has no branches or loops. run_simple_topology() assumes the first component in the list is a spout, and it passes the output of each component to the next component in the list.

License
=======

The use and distribution terms for this software are covered by the BSD 3-clause license 1.0 (http://opensource.org/licenses/BSD-3-Clause) which can be found in the file LICENSE.txt at the root of this distribution. By using this software in any fashion, you are agreeing to be bound by the terms of this license. You must not remove this notice, or any other, from this software.

Setting up Petrel
=================

First, you need to generate Python Thrift wrappers for Storm.

Make sure you have the Thrift compiler installed. From the base Petrel directory, run:

cd petrel/petrel
mkdir generated
cd generated/
thrift -gen py -out . <Path to storm.thrift>

Now run "ls" and you should see something like the following:

__init__.py  storm

Next, you need to build the jvmpetrel jar. Make sure you have Maven installed, then from the base Petrel directory, run:

cd jvmpetrel
mvn assembly:assembly

If it works, you'll see a message near the end like:

[INFO] BUILD SUCCESS

Finally, you need to set up the Petrel Python library. From the base Petrel directory, run:

cd petrel
python setup.py develop

This will download a few dependencies and then print:

Finished processing dependencies for petrel==0.0.0

Running the word count example
==============================

From the base Petrel directory, run:

cd samples/wordcount
./buildandrun --config topology.yaml

This will run the topology in local mode. You can run it on a real cluster by providing an additional parameter, the name of the topology:

./buildandrun --config topology.yaml wordcount
