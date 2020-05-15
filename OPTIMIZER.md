univocity-trader-optimizer
========================== 

The optimizer is a library built on top of the univocity-trader framework
to test and optimize your strategies quickly. The speed improvement makes it
efficient to use AI or brute force to discover optimal
parameters for one or multiple indicators in your strategy.

Once you the univocity-trader-optimizer.jar to your classpath you should be
able to import the following package:

```
import com.univocity.trading.optimizer.*;
```

Which allows you to create a market simulation with multiple symbols
traded at the same time as usual:

```
Simulator simulator = Strategy.simulator();

SimulationAccount account = simulator.configure().account();


... your usual configuration

simulation.addParameters(<your list of parameters>);
simulator.run();
```

Nothing too new here, but when running this it will produce pre-processed files
in your temp directory and allow you to run the process multiple times pretty
quickly. But that's just the beginning.

## Reusing Indicators

To make your simulations run fast it's important to reuse the state of all 
indicators that are not having changes to their parameters across each 
simulation run. For that, use the `Indicators` factory class to instantiate the
indicators in your `Strategy` or `StrategyMonitor` instead of creating
instances with the `new` keyword. 

In other words, use `Indicators.<IndicatorName>(args)` instead 
of `new <IndicatorName>(args)`.

For example, 

```
adx adx1hour = new adx(timeinterval.hours(1));
```

Should become

```
ADX adx1hour = Indicators.ADX(TimeInterval.hours(1));
```

With this, each core of your CPU can process groups of simulations, reusing
the indicators that remain the same in each simulation of the group. 

The number simulations to run per group can be configured with:

```
simulator.configure().groupSizePerThread(5);
```

Once you put this to run, you'll notice a drastic improvement in execution 
speed (from hours to seconds or minutes). 

But the real deal is the `ParameterOptimizer`, presented next

## The ParameterOptimizer

The parameter optimizer was designed to collect statistics and help to decide
which parameters produce the most relevant results for your strategy.

Notice that: 

 * The basic simulation configuration is exactly the same as before, but you
 can only use one account.

 * Each symbol will be backtested in isolation from the other symbols, using the 
parameters your provide.  

To use the optimizer, simply write:

```
ParameterOptimizer optimizer = Strategy.optimizer();
```

The optimizer configuration allows you to provide a callback for collecting
statistics:

```
optimizer.configure().collectStatistics(BiFunction<Parameters, Trade, T>);
```

Where `T` is the type of any object you define to collect data about a trade, 
with the parameters that were in use when processing it.

For example, let's create a class named `MyStatistics`:

```
public class MyStatistics {
		
	public Void processEntry(Parameters parameters, Trade trade) {
		register(
				parameters.toString(),   // e.g. Bollinger(1, 12)
				trade.symbol(),          // e.g. BTCUSDT
				trade.actualProfitLoss(),// e.g. 2.52%
				trade.maxChange(),       // e.g. 3.73%
				trade.minChange(),       // e.g. -0.13%
				trade.ticks(),           // e.g. 731
				trade.exitReason()       // e.g. stop loss
		);
		return null;
	}

    private void register(Object ... details){
		//e.g. insert details into a database
	}
}
```

Now on your config, you can use the following:

```
MyStatistics myStatistics = new MyStatistics();
optimizer.configure().collectStatistics(myStatistics::processEntry);
```

Now when the optimizer runs, every will be sent to your `MyStatistics` for 
processing.  

### Batching

The statistics collection can produce massive amounts of data pretty quickly. If
you want to use a database or some other repository of information that won't 
perform well inserting statistics one by one, you can use the in-built batch
support.

Let's modify the `MyStatistics` class to insert everything into a database:

```
public static class MyStatistics {

    public Object[] generateEntry(Parameters parameters, Trade trade) {
        return new Object[]{
                parameters.toString(),   // e.g. Bollinger(1, 12)
                trade.symbol(),          // e.g. BTCUSDT
                trade.actualProfitLoss(),// e.g. 2.52%
                trade.maxChange(),       // e.g. 3.73%
                trade.minChange(),       // e.g. -0.13%
                trade.ticks(),           // e.g. 731
                trade.exitReason()       // e.g. stop loss
        };
    }

    private static final String INSERT = "" +
            " INSERT INTO my_statistic" +
            " (parameters, symbol, profit, max_change, min_change, ticks, exitReason)" +
            " VALUES (?,?,?,?,?,?,?,?,?,?)";
    
    public void executeBatch(Queue<Object[]> queue) {
        if (queue == null) { //null queue indicats no more data.
            return;
        }
        JdbcTemplate db = getJdbcTemplate(); // Spring JDBC is great
        db.execute((ConnectionCallback<Object>) con -> {
            Object[] row = new Object[0];
            try (PreparedStatement ps = con.prepareStatement(INSERT)) {
                int len = queue.size();
                if (len > 2000) { //inserts 2K rows per batch
                    len = 2000;
                }
                for (int i = 0; i < len; i++) {
                    row = queue.poll();
                    if (row != null) {
                        for (int j = 0; j < row.length; j++) {
                            ps.setObject(j + 1, row[j]);
                        }
                        ps.addBatch();
                    } else {
                        break;
                    }
                }
                ps.executeBatch();
            } catch (Exception e) {
                log.error("Error persisting statistics: " + Arrays.toString(row), e);
            }
            return null;
        });
    }
}
```

Note that now we are returning our statistics back to the framework. It will
store it in an internal `Queue` and once it has enough data your `executeBatch`
method will be invoked.

Now you can use the following:

```
optimizer.configure().collectStatistics(myStatistics::generateEntry, myStatistics::executeBatch);
```

### Blacklisting

If you have a large list of parameters to test against multiple symbols, you can
reduce processing time by blacklisting some of these parameters during the 
optimization process.

When running, the optimizer will pick a symbol and run simulations will all 
parameters available against that symbol. Once the process is complete, all
parameters will be tested against the next symbol, and so on.

After the processing on a symbol is complete, the optimizer can query your 
statistics to ask for parameters that performed way too poorly on the previous
symbol(s) and could be discarded when processing the next. This helps reduce the
time spent running simulations for parameters that are unlikely to produce good
results for you.

We could add the following method to the `MyStatistics` class:

```
public static class MyStatistics {
    
    ...

    public Set<String> getParametersToSkip() {
        Set<String> badParameters = new HashSet<>();
        //idenfiy bad parameters based on the statistics collected
        //return the result of the .toString() of all parameters to discard  
        return badParameters;
    }
}
```

With that ready, we can configure the optimizer with: 

```
    optimizer.configure().blacklistParameters(myStatistics::getWorstParameters);
```


### Executing the optimizer.

With the statistics collection ready, we can run the optimizer with our 
parameters: 

```
optimizer.configure().simulation().addParameters(parameters);
optimizer.run();
```

... and it should do its thing, FAST.

Again, indicators that keep their parameters constant on each simulation run
will be reused in groups, and each group will run on its own thread. The 
group side and number of threads can be configured with:

```
optimizer.configure().threads(threads); //limit of threads to use
optimizer.configure().groupSizePerThread(groupSize); //defaults to 16
```
