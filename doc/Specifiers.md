# Swarm: specifying events

Swarm, the synchronized object system, addresses every event or entity by its *specifier*. A specifier is built of tokens (ids) identifying various aspects of an event/object. Ids are somewhat analogous to Mongo`s Object Ids, uniquely specifying the time of an event and its source (author). Objects are identified by the event of their creation (birth stamps).

## Identifiers

An id is approximately 12 bytes long, represented either as a 6-symbol Unicode string or  a 18-symbol Base32 string. An id contains:

- 30-bit timestamp (seconds since 1 Jan 2010)
- 15-bit sequence number to separate same-second events
- 30-bit source (author) id
- 15-bit session number to separate sessions of the same user (e.g. several browser windows)

Sometimes we use *retrofitted* ids which are simply unique strings without any internal structure. For example, class field names are encoded as ids. Their base32 representations are supposed to be human-readable. Simply random ids may also be used sometimes.

## Specifiers

Ids are *words* of our addressing system. Specifiers are meaningful *sentences* having their nouns, verbs and adjectives. To distinguish those we use "quants", one-char special symbols that provide the context for an id that follows. Valid quants are /#.,:!$. For example,

    /ClassName#object_id.fieldName

points at a particular field of a particular object and

    /ClassName#object_id!rev_id

addresses a particular revision of that object.
Formally, specifiers are tuples of quant-prefixed ids.
Specifier is an extremely handy formalism for the purpose of data/event storage/serialization. It provides non-ambiguous addressing and nearly-binary storage efficiency. The Swarm pumps data around as a stream of small incremental events, as opposed to e.g. bulky JSON objects. Specifiers are both the enabler and an optimization of that process.

## Forms: raw and parsed

A specifier in its raw form is a Unicode string consisting of 7-symbol tokens. Each token is a quant followed by 6-symbol id. If you know what I mean:

    /([\/\#\.\,\:\!\$])([0-\u802f]{6})/g

That form is compact and simple, albeit not that human readable. Generally, specifiers are parsed on demand to avoid unnecessary work. The next stage after a string is a semi-parsed object:

    {
        collection,  //  /
        object,      //  #
        field,       //  .
        key,         //  ,
        method,      //  :
        version,     //  !
        base,        //  $
        cache        // the original specifier string
    }
Each field is the corresponding id of the specifier represented as a 7-symbol Unicode string or an empty string if there is none. In some special cases a field may contain multiple concatenated ids.
Then, every id may be parsed further into an object:

    {
        q,   // quant int
        ts,  // timestamp int
        seq, // sequence number int
        src, // source int
        ssn, // session int
        cache,  // the id string
        cache32 // base32 id string
    }

Capitalization of the Base32 string depends on the quant. CollectionNames are given in CamelCase, methodNames in lowerCamelcase, other ids are undrscore_separated. Note that RFCXXX Base32 lacks '1', '2' and '0'. Field names and class names must be valid ids in the Base32 form or they are not picked up by Swarm and not synchronized.
Both Spec and ID objects have the usual toString() method returning the compact Unicode representation. Its twin toString32() method returns the Base32 representation. It is mostly useful for debugging as Base32 strings are (simply) three times longer.
Parsed objects are (generally) supposed to be immutable. In case you modify such an object please nullify its cache field or otherwise toString will return stale data. 

## API

Three core methods Swarm installs on every synched object are:

    on(spec,fn)   // start listening to an object
    set(spec,val) // set field value
    off(spec,fn)  // stop listening

Other methods are:

    once(spec,fn)     // listen to an event once
    diff(spec,obj)    // check for changes
    get(spec,default) // get field value
    rev(spec,i)       // get current revision id

Note that the first argument is always a specifier. We did our best to generalize the popular on/set/off convention so

    var spec = new Spec();
    spec.field = 'x';
    obj.get('x',1) === obj.get(spec,1)  // true

and generally a string provided as the first argument is considered to be some id in the Base32 form unless it matches the specifier regex (starts with a quant etc).
Similarly, Swarm provides convenience methods for every field detected on a new object, like

    Swarm.extend(SampleObject);
    var obj = new SampleObject(); // has x
    obj.setX(1) === 1;
    obj.getX() === obj.get('x');

Callback functions use the same two arguments:

    obj.on('x',function(spec,val){
        console.log(val);   //  outputs '1'
    }).setX(1);

It is possible to do "bundled" set calls:

    obj.set('',{x:1,y:2}) // '' counts as a spec

The usual convention is also supported:

    obj.set({x:1,y:2})  // anon object isnt a Spec

The bundled set call is even preferred as it generates one version while a sequence of set calls generates a sequence of versions.

Swarm: sync`em all!