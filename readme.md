# NAME

Log::Any

# SYNOPSIS

```perl6
use Log::Any;
use Log::Any::Adapter::File;
Log::Any.add( Log::Any::Adapter::File.new( '/path/to/file.log' ) );

Log::Any.info( 'yolo' );
Log::Any.error( :category('security'), 'oups' );
Log::Any.log( :msg('msg from app'), :category( 'network' ), :severity( 'info' ) );
```

# DESCRIPTION

Log::Any is a library to generate and handle application logs.
A log is a message indicating an application status at a given time. It has attributes, like a _severity_ (error, warning, debug, …), a _category_, a _date_ and a _message_.

These attributes are used by the "Formatter" to format the log and can also be used to filter logs and to choose where the log will be handled (via Adapters).

## SEVERITY

The severity is the level of urgence of a log.
It can take the following values (based on Syslog):
- emergency
- alert
- critical
- error
- warning
- notice
- info
- debug
- trace

TODO:
Idealy, the severity levels should be specifiable while creating the log system, because some users would need some other severity order.
```perl6
Log::Any.severities( [ 'level1', 'level2', … ] );
```

## CATEGORY

The category can be seen as a group identifier.

Ex:
- security ;
- database ;
- ...

_Default value_ : the package name where the log is generated.

## DATE

The date is generated by Log::Any, and its values is the current date and time (ISO 8601).

## MESSAGE

A message is a string passed to Log::Any defined by the user.
TODO: Proxies to log more than a string (or use formatter?).

# ADAPTERS

An adapter handles a log by storing it, or sending it elsewhere.
If no adapters are defined, or no one meets the filtering, the message will not be logged.

A few examples:

- Log::Any::Adapter::File
- Log::Any::Adapter::Database::SQL
- Log::Any::Adapter::STDOUT

## Provided adapters

### File

```perl6
use Log::Any::Adapter::File;
Log::Any.add( Log::Any::Adapter::File.new( path => '/path/to/file.log' ) );
```

### Stdout

```perl6
use Log::Any::Adapter::Stdout;
Log::Any.add( Log::Any::Adapter::Stdout.new );
```

### Stderr

```perl6
use Log::Any::Adapter::Stderr;
Log::Any.add( Log::Any::Adapter::Stderr.new );
```

# FORMATTERS

Often, logs need to be formatted to simplify the storage (time-series databases), or the analysis (grep, log parser).

Formatters will use the attributes of a Log.

|Symbol|Signification|Description                                  |Default value             |
|------|-------------|---------------------------------------------|--------------------------|
|\\d   |Date (UTC)   |The date on which the log was generated      |Current date time         |
|\\c   |Category     |Can be any anything specified by the user    |The current package/module|
|\\s   |Severity     |Indicates if it's an information, or an error| none                     |
|\\m   |Message      |Payload, explains what is going on           | none                     |

```perl6
use Log::Any::Adapter::STDOUT( :formatter( '\d \c \m' ) );
```

You can of course use variables in the formatter, but since _\\_ is already used in Perl6 strings interpolation, you have to escape them.

```perl6
my $prefix = 'myapp ';
use Log::Any::Adapter::STDOUT( :format( "$prefix \\d \\c \\s \\m" ) );
```

A formatter can be more complex than the default one by extending the class _Formatter_.

```perl6
use Log::Any::Formatter;

class MyOwnFormatter is Log::Any::Formatter {
	method format( :$dateTime!, :$msg!, :$category!, :$severity! ) {
		# Returns an Str
	}
}
```

TODO:
An adapter can define a prefered formatter which will be used if no formatter are specified.

# FILTERS

Filters can be used to allow a log to be handled by an adapter.
Many fields can be filtered, like the category, the severity or the message.

The easiest way to define a filter is by using the _filter_ parameter of Log::Any.log method.

```perl6
Log::Any.add( Adapter.new, :filter( [ <filters fields goes here>] ) );
```

Filtering on category or message:
```perl6
# Matching by String
Log::Any.add( Adapter.new, :filter( ['category' => 'My::Wonderfull::Lib' ] ) );
# Matching by Regex
Log::Any.add( Adapter.new, :filter( ['category' => /My::*::Lib/, 'severity' => '>warning' ] ) );

# Matching msg by Regex
Log::Any.add( Adapter.new, :filter( [ 'msg' => /a regex/ ] );
```

Filtering on severity:

The severity can be considered as levels, so can be traited as numbers.

1. trace
2. debug
3. info
4. notice
5. warning
6. error
7. critical
8. alert
9. emergency

Filtering on severity can be done by specifying an operator in front of the severity:

```perl6
filter => [ 'severity' => '>warning' ] # Above
filter => [ 'severity' => '==debug'  ] # Equality
filter => [ 'severity' => '<notice'  ] # Beside
```

Matching only several severities is also possible:

```perl6
filter => [ 'severity' => [ 'notice', 'warning' ] ]
```

TODO: matching all but a list of severities:

```perl6
filter => [ 'severity' => [ * - 'warning' ] ] # All but 'warning'
```

If several filters are specified, all must be valid

```perl6
# Use this adapter only if the category is My::Wonderfull::Lib and if the severity is warning or error
[ 'severity' => '>=warning', 'category' => /My::Wonderfull::Lib/ ]
```

If a more complex filtering is necessary, a class can be created:
```perl6
# Use home-made filters
class MyOwnFilter is Log::Any::Filter {
	method filter( :$msg, :$severity, :$category ) returns Bool {
		# Write some complicated tests
		return True;
	}
}

Log::Any.add( Some::Adapter.new, :filter( MyOwnFilter.new ) );
```

## Filters acting like barrier

/!\ Work in progress /!\

```perl6
Log::Any.add( Adapter.new );
Log::Any.add( :filter( [ severity => '>warning' ] );
# Only logs with severity above _warning_ continues through the pipeline.
Log::Any.add( Adapter2.new );
```

# PIPELINES

A _pipeline_ is a set of Adapters and can be used to define alternatives path (a set of adapters, filters, formatters and options (asynchronicity) ). This allows to handle differently some logs (for example, for security or realtime).
If a log is produced with a specific pipeline which is not defined in the log consumers, the default pipeline is used.

Pipelines can be specified when an Adapter is added.
```perl6
Log::Any.add( :pipeline('security'), Log::Any::Adapter::Example.new );

Log::Any.error( :pipeline('security'), :msg('security error!', ... ) );
```

# INTROSPECTION

Check if a log will be handled (to prevent computation of log).
It is usefull if you want to log a dump of a complex object which can take time.

## will-log method

This method can takes in parameters the *category*, the *severity* and the *pipeline* to use.
Theses parameters are then used to check if the message could pass through the pipelines and will be handled.

/!\ *msg* parameter cannot be tested, so if a filter acts on it, the results will differ between will-log() and log().  /!\

```perl6
if Log::Any.will-log( :severity('debug') ) {
	Log::Any.debug( serialize( $some-complex-object ) );
}
```

## will-log aliases

Some aliases are defined and provide the severity.

- will-emergency()
- …
- will-trace()

```perl6
Log::Any.will-debug(); # Alias to will-log( :severity('debug') )
```

# DEBUGGING

## gist method

_gist_ method prints a string representing the internal Log::Any state (defined pipelines with their adapters, filters and formatters). Since many attributes are not public, you cannot recreate a Log::Any stack based on this representation.

# EXTRA FEATURES

## Exporting aliases

Can be usefull to use more consise routines :

```perl6
use Log::Any( :subs );

log-adapt( Adapter.new );

warning( 'missing some configuration' );
critical( 'a big problem occured' );
```

## Wrapping

### STDOUT, STDERR

Sometimes, applications or libraries are already available, and prints their logs to STDOUT and/or STDERR. Log::Any could captures these logs and manage them.

### EXCEPTIONS

Catches all unhandled exceptions to log them.

## Stacktrace

Dump a stacktrace with the log. This could be usefull to find a problem.
	Is it necessary?
	Is it possible to do in an Adapter or some Proxy ?

## Asynchronicity

```perl6
use Log::Any ( :async );

Log::Any.pipeline( 'myapp' ).async( True );
```

## log-on-error

keep in cache logs in streams (all, from trace to info)
	- if an error occurs (how to detect, using a level?), log the stacktrace ;
	- if nothing special occurs, log cached logs as specified in the filters.

## Load log configuration from configuration files

- with a watcher on the file ?
	- pause dispatching during the reload ;

## PROXYs

A proxy is a class used to intercept messages before they are relly sent to the log subroutine. They can be usefull to log more than strings, or to analyse the message. They can also add some data in the message like tags.
	todo: is a filter, a proxy?

## TAGS

Where?
	- in place of category ?
	- as extended informations ? +1
How?
	- tags: [ tag1, tag2 ]
	- how to log them (array) ?

## Die on error

Returns an error (logging, exception?) if a log is not handled.

## Does not log duplicate messages

Check if a log message already been logged during the last <timespec>.

Will call the proxy before logging the message.
	- if the message already been seen n times during the last 1s, increment a counter and return false (meaning the log will not be logged).
		- a timer executing every n s will print a message like '$msg has been seen n times during the last n s' ;
			- empty the stack
	- if not, log basically the message.

```perl6
Log::Any.add( $adapter, :proxy( Log::Any::Proxy::CacheUnpoisonning.new( :stack-size( 10 ), :time-interval( '1s' ) );
```

Prints something like:
```
Wow an error in an infinite loop...
* Log::Any : previous message reapeated n times during last 1s.
```

