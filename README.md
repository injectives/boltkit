# Boltkit

Boltkit is a collection of tools and resources for Neo4j 3.0+ driver authors.

**Contents**

- [Installation](#installation)
- [Demo Driver](#demo-driver)
- [Statement Runner](#statement-runner)
- [Stub Bolt Server](#stub-bolt-server)
  - [Scripting](#stub-bolt-server/scripting)
  - [Command Line Usage](#stub-bolt-server/command-line-usage)
  - [Java Test Usage](#stub-bolt-server/java-test-usage)
- [Neo4j Controller](#neo4j-controller)
- [Single Server](#neo4j-controller)
  - [`neoctrl-download`](#neo4j-controller/download)
  - [`neoctrl-install`](#neo4j-controller/install)
  - [`neoctrl-start`](#neo4j-controller/start)
  - [`neoctrl-stop`](#neo4j-controller/stop)
  - [`neoctrl-uninstall`](#neo4j-controller/uninstall)
  - [`neoctrl-create-user`](#neo4j-controller/create-user)
  - [`neoctrl-configure`](#neo4j-controller/configure)
- [Cluster](#neo4j-controller/cluster)
  - [`neoctrl-cluster install`](#neo4j-controller/cluster-install)
  - [`neoctrl-cluster start`](#neo4j-controller/cluster-start)
  - [`neoctrl-cluster stop`](#neo4j-controller/cluster-stop)
  - [`neoctrl-cluster uninstall`](#neo4j-controller/cluster-uninstall)
----


## <a name="installation"></a>Installation

The package can either be installed globally or within a *virtualenv*.

Installation makes available several command line tools, the names of which all start with either `bolt` or `neoctrl`.

```
pip install --upgrade boltkit
```

Install into user space for python 3:
```
python3 -m pip install --user --upgrade boltkit
```

Check installed version on user space for python 3:
```
python3 -m pip freeze --user
```

Install into user space for python 3 directly from github repo:
```
python3 -m pip install --user git+https://github.com/neo4j-drivers/boltkit@1.3#egg=boltkit
```

Install into user space for python 3 directly from local clone with edit mode:
```
python3 -m pip install --editable --user <path/to/neo4j-drivers/boltkit>
```

## <a name="demo-driver"></a>Demo Driver 

- Source: [`boltkit/driver.py`](boltkit/driver.py)

This file contains both a fully-working Neo4j driver as well as a step-by-step tutorial for how to implement a driver in any language.
To view the code and tutorial in a terminal, use:

```
less $(python -c "from boltkit import driver; print(driver.__file__)")
```


## <a name="statement-runner"></a>Statement Runner

- Command: `boltrun <statement>`
- Source: [`boltkit/runner.py`](boltkit/runner.py)

Example:
```
boltrun "UNWIND range(1, 10) AS n RETURN n"
```


## <a name="stub-bolt-server"></a>Stub Bolt Server 

- Command: `boltstub <port> <script>`
- Source: [`boltkit/server.py`](boltkit/server.py)

The stub Bolt server can be used as a testing resource for client software.
Scripts can be created against which unit tests can be run without the need for a full Neo4j server.

A server script describes a conversation between client and server in terms of the messages
exchanged. An example can be seen in the [`test/scripts/count.bolt`](test/scripts/count.bolt) file.

When the server receives a client message, it will attempt to match that against the next client message in the script; if found, this line of the script will be consumed.
Then, any server messages that follow will also be consumed and sent back.
When the client closes its connection, the server will shut down.
If any script lines remain, the server will exit with an error status; if none remain it will exit successfully.
After 30 seconds of inactivity, the server will time out and shut down with an error status.

### <a name="stub-bolt-server/scripting"></a>Scripting 

Scripts generally consist of alternating client (`C:`) and server (`S:`) messages.
Each message line contains the message name followed by its fields, in JSON format.

Some messages, such as `INIT` and `RESET`, can be automatically (successfully) consumed if they are not relevant to the current test.
For this use a script line such as `!: AUTO INIT`.

An example:
```
!: AUTO INIT
!: AUTO RESET

C: RUN "RETURN {x}" {"x": 1}
   PULL_ALL
S: SUCCESS {"fields": ["x"]}
   RECORD [1]
   SUCCESS {}
```


### <a name="stub-bolt-server/command-line-usage"></a>Command Line Usage 

To run a stub server script:
```
boltstub 7687 test/scripts/count.bolt
```

To run a Cypher command against the stub server:
```
boltrun "UNWIND range(1, 10) AS n RETURN n"
```

### <a name="stub-bolt-server/java-test-usage"></a>Java Test Usage 

The stub server can be used from any environment from which command line tools can be executed.
To use from Java, first construct a wrapper for the server:
```java
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import static java.lang.Thread.sleep;
import static java.util.Arrays.asList;
import static java.util.Collections.singletonList;

public class StubServer
{
    // This may be thrown if the driver has not been closed properly
    public static class ForceKilled extends Exception {}

    private Process process = null;

    private StubServer( String script, int port ) throws IOException, InterruptedException
    {
        List<String> command = new ArrayList<>();
        // This assumes the `boltstub` command is available on the path
        command.addAll( singletonList( "boltstub" ) );
        command.addAll( asList( Integer.toString( port ), script ) );
        ProcessBuilder server = new ProcessBuilder().inheritIO().command( command );
        process = server.start();
        sleep( 500 );  // might take a moment for the socket to start listening
    }

    public static StubServer start( String script, int port ) throws IOException, InterruptedException
    {
        return new StubServer( script, port );
    }

    public int exitStatus() throws InterruptedException, ForceKilled
    {
        sleep( 500 );  // wait for a moment to allow disconnection to occur
        try
        {
            return process.exitValue();
        }
        catch ( IllegalThreadStateException ex )
        {
            // not exited yet
            process.destroy();
            process.waitFor();
            throw new ForceKilled();
        }
    }

}
```

Then, assuming you have a valid script available, create a test to use the stub:
```java
@Test
public void shouldBeAbleRunCypher() throws StubServer.ForceKilled, InterruptedException, IOException
{
    // Given
    StubServer server = StubServer.start( "/path/to/resources/return_x.bolt", 7687 );
    URI uri = URI.create( "bolt://localhost:7687" );
    int x;

    // When
    try ( Driver driver = GraphDatabase.driver( uri ) )
    {
        try ( Session session = driver.session() )
        {
            Record record = session.run( "RETURN {x}", parameters( "x", 1 ) ).single();
            x = record.get( 0 ).asInt();
        }
    }

    // Then
    assertThat( x, equalTo( 1 ) );

    // Finally
    assertThat( server.exitStatus(), equalTo( 0 ) );
}
```

## <a name="neo4j-controller"></a>Neo4j Controller 

The Neo4j controller module comprises a set of scripts for downloading, starting, stopping and configuring Neo4j servers.
These scripts should work for any 3.0+ server version and can pull artifacts from local file system, TeamCity and AWS server (credentials might be required).

Module also contains `neoctrl-cluster` and `neoctrl-multicluster` commands that give ability to install, start and stop Neo4j Causal clusters and Neo4j Multi-clusters.
Neo4j cluster commands support 3.1+ enterprise Neo4j versions only.
And Neo4j multi-cluster commands support 3.4+ enterprise Neo4j versions only.

### <a name="neo4j-controller/single-instance"></a>Standalone Server

#### `neoctrl-download` <a name="neo4j-controller/download"></a>
```
usage: neoctrl-download [-h] [-e] [-v] version [path]

Download a Neo4j server package for the current platform.

example:
  neoctrl-download -e 3.1.0-M09 $HOME/servers/
  neoctrl-download -e 3.3 $HOME/servers/

positional arguments:
  version           Neo4j server version
  path              download destination path (default: .)

optional arguments:
  -h, --help        show this help message and exit
  -e, --enterprise  select Neo4j Enterprise Edition (default: Community)
  -v, --verbose     show more detailed output

environment variables:
  DIST_HOST         name of distribution server (default: dist.neo4j.org)
  TEAMCITY_HOST     name of build server
  TEAMCITY_USER     build server user name
  TEAMCITY_PASSWORD build server password
  AWS_ACCESS_KEY_ID aws access key id
  AWS_SECRET_ACCESS_KEY aws secret access key
  NEOCTRL_LOCAL_PACKAGE local package source is always preferred if configured


TEAMCITY_* environment variables are required to download snapshot servers.
AWS_* environment variables are used to access enterprise distribution servers.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/install"></a>`neoctrl-install`
```
usage: neoctrl-install [-h] [-e] [-v] version [path]

Download and extract a Neo4j server package for the current platform.

example:
  neoctrl-install -e 3.1.0-M09 $HOME/servers/

positional arguments:
  version           Neo4j server version
  path              download destination path (default: .)
  service_name      set windows service name if specified (optional)

optional arguments:
  -h, --help        show this help message and exit
  -e, --enterprise  select Neo4j Enterprise Edition (default: Community)
  -v, --verbose     show more detailed output

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/start"></a>`neoctrl-start`
```
usage: neoctrl-start [-h] [-v] [home]

Start an installed Neo4j server instance.

example:
  neoctrl-start $HOME/servers/neo4j-community-3.0.0

positional arguments:
  home           Neo4j server directory (default: .)

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  show more detailed output

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/stop"></a>`neoctrl-stop`
```
usage: neoctrl-stop [-h] [-v] [-k] [home]

Stop an installed Neo4j server instance.

example:
  neoctrl-stop $HOME/servers/neo4j-community-3.0.0

positional arguments:
  home           Neo4j server directory (default: .)

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  show more detailed output
  -k, --kill     forcefully kill the instance

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/uninstall"></a>`neoctrl-uninstall`
```
usage: neoctrl-uninstall [-h] [-v] [home]

Uninstall a Neo4j server instance.

example:
  neoctrl-uninstall $HOME/servers/neo4j-community-3.0.0

positional arguments:
  home           Neo4j server directory (default: .)

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  show more detailed output

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/create-user"></a>`neoctrl-create-user`
```
usage: neoctrl-create-user [-h] [-v] [home] user password

Create a new Neo4j user.

example:
  neoctrl-create-user $HOME/servers/neo4j-community-3.0.0 bob s3cr3t

positional arguments:
  home           Neo4j server directory (default: .)
  user           name of new user
  password       password for new user

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  show more detailed output

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/configure"></a>`neoctrl-configure`
```
usage: neoctrl-configure [-h] [-v] [home] key=value [key=value ...]

Update Neo4j server configuration.

example:
  neoctrl-configure . dbms.security.auth_enabled=false

positional arguments:
  home           Neo4j server directory (default: .)
  key=value      key/value assignment

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  show more detailed output

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/set-initial-password"></a>`neoctrl-set-initial-password`
```
usage: neoctrl-set-initial-password [-h] password [home]

Sets the initial password of the initial admin user ('neo4j').

example:
  neoctrl-set-initial-password newPassword $HOME/servers/neo4j-community-3.0.0

positional arguments:
  password    password for the admin user
  home        Neo4j server directory (default: .)

optional arguments:
  -h, --help  show this help message and exit

Report bugs to drivers@neo4j.com
```

### <a name="neo4j-controller/cluster"></a>Cluster
```
usage: neoctrl-cluster [-h] {install,start,stop} ...

Operate Neo4j causal cluster.

optional arguments:
  -h, --help            show this help message and exit

available sub-commands:
  start                 Start the causal cluster located at the given path
  stop                  Stop the causal cluster located at the given path
  install               Download, extract and configure causal cluster

  {install,start,stop}  commands are available

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/cluster-install"></a>`neoctrl-cluster install`
```
usage: neoctrl-cluster install [-h] [-v] [-c CORE_COUNT]
                               [-r READ_REPLICA_COUNT] [-i INITIAL_PORT]
                               -p PASSWORD
                               version [path]

Download, extract and configure causal cluster

example:
  neoctrl-cluster install [-v] [-c 3] -p init_password 3.1.0 $HOME/cluster/

positional arguments:
  version               Neo4j server version
  path                  download destination path (default: .)

optional arguments:
  -h, --help            show this help message and exit
  -v, --verbose         show more detailed output
  -c CORE_COUNT, --cores CORE_COUNT
                        number of core members in the cluster (default 3)
  -r READ_REPLICA_COUNT, --read-replicas READ_REPLICA_COUNT
                        number of read replicas in the cluster (default 0)
  -i INITIAL_PORT, --initial-port INITIAL_PORT
                        initial port number for all used ports on all cluster
                        members. Each next port will simply be an increment of
                        the previous one (default 20000)
  -p PASSWORD, --password PASSWORD
                        initial password of the initial admin user ('neo4j')
                        for all cluster members (always required)

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/cluster-start"></a>`neoctrl-cluster start`
```
usage: neoctrl-cluster start [-h] [-t TIMEOUT] [path]

Start the causal cluster located at the given path

example:
  neoctrl-cluster start $HOME/cluster/

positional arguments:
  path                  causal cluster location path (default: .)

optional arguments:
  -h, --help            show this help message and exit
  -t TIMEOUT, --timeout TIMEOUT
                        startup timeout in seconds (default: 120)

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/cluster-stop"></a>`neoctrl-cluster stop`
```
usage: neoctrl-cluster stop [-h] [-k] [path]

Stop the causal cluster located at the given path

example:
  neoctrl-cluster stop $HOME/cluster/

positional arguments:
  path        causal cluster location path (default: .)

optional arguments:
  -h, --help  show this help message and exit
  -k, --kill  forcefully kill all instances in the cluster

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```

#### <a name="neo4j-controller/cluster-uninstall"></a>`neoctrl-cluster uninstall`
```
usage: neoctrl-cluster uninstall [-h] [path]

Uninstall the causal cluster located at the given path

example:
  neoctrl-cluster uninstall $HOME/cluster/

positional arguments:
  path                  causal cluster location path (default: .)

optional arguments:
  -h, --help            show this help message and exit

See neoctrl-download for details of supported environment variables.

Report bugs to drivers@neo4j.com
```
