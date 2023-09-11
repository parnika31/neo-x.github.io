---
layout: post
title: 'plpgsql: Creating Functions and saving intermetiate query results'
date: '2013-03-03T11:39:00.002-08:00'
author: Glen B
tags: 
modified_time: '2013-03-06T08:19:25.205-08:00'
blogger_id: tag:blogger.com,1999:blog-2030049895335802245.post-3043489675023951258
blogger_orig_url: http://fracturedplane.blogspot.com/2013/03/plpgsql-creating-functions-and-saving.html
---

<br />Sometimes SQL queries are difficult or impossible to optimize if you are using pure standard SQL.<br /><br />I started this because I just wanted to find a way to save the result of a sub-query to use later. In my example I wanted to save the average&nbsp;+ the standard deviation  of a large set of values for some research. I found that Posgresql supports standard SQL only so you can not save query results. Instead you can use plpgsql with is a procedural langauge that can be used with Postgresql (psql).<br /><br />These are the two tables being considered for these examples.<br /><br /><pre class="prettyprint"><code><br />CREATE TABLE IF NOT EXISTS test <br /><br />(<br /><br />&nbsp; test_id integer NOT NULL primary key,<br /><br />&nbsp; algorithm_data_id integer NOT NULL references <br /><br />&nbsp; test_timestamp timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,<br /><br />&nbsp; test_status integer NOT NULL,<br /><br />&nbsp; num_agents integer NOT NULL,<br /><br />&nbsp; num_obstacles integer NOT NULL<br /><br />) ;<br /><br /><br /><br />CREATE TABLE IF NOT EXISTS composite1<br /><br />&nbsp;&nbsp;&nbsp; benchmark_id integer NOT NULL references test(test_id) ON DELETE CASCADE,<br /><br />&nbsp;&nbsp;&nbsp; collisions integer NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; time double precision NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; success integer NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; num_unique_collisions integer NOT NULL,&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; total_distance_travelled double precision NOT NULL,&nbsp;&nbsp;&nbsp; &nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; agent_complete integer NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; agent_success integer NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />&nbsp;&nbsp;&nbsp; optimal_path_length double precision NOT NULL,&nbsp;&nbsp;&nbsp; <br /><br />);<br /><br /></code></pre><br />Basic syntax for a function definition<br /><br /><pre class="prettyprint"><code><br />CREATE OR REPLACE FUNCTION multiply(x real, y real) RETURNS real AS&nbsp;<br /></code></pre>CREATE OR REPLACE The replace part is useful for development if testing a function and updating it but also useful if it so happens that there already exist a function with the same name you want to override it to ensure proper functionality.<br /><br />After the AS comes the function definition. The most non obvious part that I found was the use of '$$'. Think of the $$ as {} from many programming languages that are used to symbolize a body of code, in this case the function body.<br /><br /><pre class="prettyprint"><code><br /><pre class="PROGRAMLISTING">CREATE OR REPLACE FUNCTION multiply(x real, y real) RETURNS real AS</pre><pre class="PROGRAMLISTING">$$<br />BEGIN<br />    RETURN x * y;<br />END;<br />$$ LANGUAGE plpgsql;<br /></pre></code></pre><pre class="PROGRAMLISTING">&nbsp;</pre><pre class="PROGRAMLISTING">This is a simple example of a simple definition that can be used to multiply two numbers.</pre><pre class="PROGRAMLISTING">&nbsp;</pre>Note the uses of ';' they are only at the end of return statements and queries. This will be seen more later.<br /><br />A more advanced example:<br /><br /><pre class="prettyprint"><code><br />CREATE OR REPLACE FUNCTION getAVG_WRTTotalTicksforScenario(scenarioID INT) RETURNS double precision AS<br /><br />$BODY$<br /><br />DECLARE<br /><br />average double precision;<br /><br />BEGIN<br /><br />&nbsp;&nbsp;&nbsp; <br /><br />SELECT AVG(total_total_ticks_accumulated/total_number_of_times_executed) INTO STRICT average<br /><br />&nbsp;&nbsp;&nbsp; FROM ppr_ai_data pd2, test test, algorithm_data<br /><br />&nbsp;&nbsp;&nbsp; WHERE test.algorithm_data_id = algorithm_data.algorithm_data_id and<br /><br />&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; -- test.scenario_group = scenario_group and<br /><br />&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; algorithm_data.algorithm_data_id = pd2.ppr_ai_id and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; test.scenario_group = scenarioID;<br /><br />&nbsp;&nbsp;&nbsp; RETURN average;<br /><br />END<br /><br />$BODY$ LANGUAGE plpgsql; <br /></code></pre><br /><br />Any variables that are going to be used should be defined in the declare section.<br /><br />The return types of functions can be of the basic types for can be of a time defined for a table. For example<br /><br /><br /><pre class="prettyprint"><code><br />CREATE OR REPLACE FUNCTION getAllSignificantTestsForScenario(scenarioID INT, algorithmName TEXT) RETURNS SETOF Test AS<br /><br />$BODY$<br /><br />DECLARE<br /><br />average double precision; <br /><br />std_dev double precision;<br /><br />r test%rowtype;<br /><br /><br /><br /><br /><br />BEGIN<br /><br /><br /><br />average := getAVG_WRTTotalTicksforScenario(scenarioID)<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; <br /><br />std_dev :=&nbsp;&nbsp;&nbsp; getSTDDEV_WRTTotalTicksforScenario(scenarioID)<br /><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; <br /><br />RAISE NOTICE 'Average is %', average;<br /><br />RAISE NOTICE 'Standard deviation is %', std_dev;<br /><br /><br /><br />RETURN QUERY &nbsp;&nbsp;&nbsp; SELECT * from test where test_id IN (<br /><br />SELECT t.test_id<br /><br />&nbsp;&nbsp;&nbsp; FROM test t, ppr_ai_data pd, algorithm al, algorithm_data Adata<br /><br />&nbsp;&nbsp;&nbsp; WHERE t.algorithm_data_id = Adata.algorithm_data_id and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; pd.ppr_ai_id = Adata.algorithm_data_id and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; Adata.algorithm_type = al.algorithm_id and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; al.name = algorithmName and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; t.scenario_group = scenarioID and<br /><br />&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; pd.total_total_ticks_accumulated / pd.total_number_of_times_executed &gt; (average + std_dev));<br /><br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; <br /><br />RETURN;<br /><br /><br /><br />END<br /></code></pre><br /><br />Return can be used a in few ways. In the above case it is used to return the result of an entire query. In the above example the return is typed to that of a row of the table test. A small trick was used to get around having to return a full test row. The use of Raise in this case is just used to inform the user of the values being used in the sub query. Usually RAISE is used to inform the user issues in the function and can be used to throw exception like many other programming languages [4].<br /><br />References<br /><ol><li>http://www.postgresql.org/docs/9.1/static/plpgsql-statements.html</li><li>http://www.postgresql.org/docs/8.4/static/plpgsql-declarations.html</li><li>http://www.postgresql.org/docs/9.1/static/plpgsql-control-structures.html#PLPGSQL-STATEMENTS-RETURNING</li><li>http://www.postgresql.org/docs/9.1/static/plpgsql-errors-and-messages.html</li></ol>