---
layout: post
title: C++ Example
image: https://www.sharkbite.io/wp-content/uploads/2017/02/sharkbite.jpg
---
<p>The following is an example of using the API to create a scanner and read data from an accumulo instance.</p>

{% highlight c++ linenos %}

    //This code shows an example of reading data from an Accumulo instance.

    //TODO unsure about assignments below

    string instancestr = argv[1];

    string zookeepers = argv[2];

    string username = argv[3];

    string password = argv[4];



    auto instance = new ZookeeperInstance(instancestr, zookeepers, 1000);



    AuthInfo creds(username, password, instance->getInstanceId());



    auto master = new MasterConnect(&amp;creds, instance);



    auto ops = master->tableOps(table);

    // create the scanner with ten threads.

    auto scanner = ops->createScanner (&amp;auths, 10);

    // range from a to d

    Key startkey;

    startkey.setRow("a", 1);

    Key stopKey;

    stopKey.setRow("d", 1);

    Range range(startkey, true, stopKey, true); 

    // build your range.

    scanner->addRange(&amp;range);



    auto results = scanner->getResultSet ();



    for (const auto &amp;iter : results) {

	auto kv = *iter;

	if (kv != NULL &amp;&amp; kv->getKey() != NULL)

	    cout << "got -- " << (*iter)->getKey() << endl;

	else

	    cout << "Key is null" << endl;

    }
{% endhighlight %}




<br />
<p>older variant</p>
{% highlight c++ linenos %}

    //This code shows an example of reading data from an Accumulo instance.



    //TODO unsure about assignments below

    string instance = argv[1];

    string zookeepers = argv[2];

    string username = argv[3];

    string password = argv[4];



    ZookeeperInstance *instance = new ZookeeperInstance(instance, zookeepers, 1000);



    AuthInfo creds(username, password, instance->getInstanceId());



    interconnect::MasterConnect *master = new MasterConnect(&amp;creds, instance);



    std::unique_ptr<interconnect::AccumuloTableOperations> ops = master-&gt;tableOps(

	    table);

    // create the scanner with ten threads.

    std::unique_ptr<scanners::BatchScanner> scanner = ops-&gt;createScanner (&amp;auths, 10);

    // range from a to d

    Key startkey;

    startkey.setRow("a", 1);

    Key stopKey;

    stopKey.setRow("d", 1);

    Range *range = new Range(startkey, true, stopKey, true); 

    // build your range.

    scanner->addRange(range);



    scanners::Iterator<cclient::data::KeyValue> *results =

			scanner->getResultSet ();



    for (auto iter : results) {

	KeyValue *kv = *iter;

	if (kv != NULL &amp;&amp; kv->getKey() != NULL)

	    cout << "got -- " << (*iter)->getKey() << endl;

	else

	    cout << "Key is null" << endl;

    }
{% endhighlight %}

