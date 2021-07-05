### How it works

 1. The library keep track of the execution order with two counters. One is increased every time mb_query is called [qid_cnt], and the other counter [qid] is increased when a query finishes, with or without error. If mb_query is called and the two counters are equal, then the library know it's that particular queries turn to be executed.

 2. When a query executed, it will start by determined the length of the query, by counting the number of bytes of data_ptr. This in one of many nice features of the library.

 3. Then the query will start with transfer its parameter to a internal memory. If it's a write query it will also serialize and transfer the content of data_ptr, into an internal buffer. When this is done, a flag is set indicating that a new query is ready to be processed.

 4. The «ctrl», (mb_rtu1200_ctrl, mb_rtu1500_ctrl or mb_tcp_ctrl) is the FB that contain the modbus blocks that come along with TIA-portal. The FB see that a query is ready and it will process it. This may take many plc-scans, when the result come back from the modbus device, the controller will send a message to the query about the arriving result.

 5. The query who inserted the query, gets the message about the finished resualt. To be sure that the right query recive the resualt, the internal query parameters is compared to current query parameters. 

 6. If it's a read query it will deserialize the buffer and transfer the content into data_ptr.

 7. The second counter [qid] will be increased, and the next query will be executed starting from step one.

 8. When all queries has been carried out, the first counter [qid] will be reseted back to the first query, and the process will repeat itself. The code that resetting of the counters is located inside the FB mb_internal_e.


### Queries execution order

In general the user of the library should not need to worry about the execution order, however for curious user there is a is a brif overview bellow on how it works. There are many if's and but's, about the order, each step bellow introduce and explains more and more complexity. 

1. If the there is no mb_station_header and mb_station_footer, no if-statements, then the execution is linear. The 1st query execute first, then the 2nd, and so on. After the last query the proccess will start over.

2. If one or more queries are surrounded by a if-statement, then the order of execution may be interupted. The reason for this is becouse query-id the queries occupies, will be given to other queries when the if-statment change state. There is however no change that a resualt from query should end in a other query. Only queries after the if-statment will be affected.    

3. With the mb_station_block_header, and mb_station_footer the complexity increase quite a lot. This FC's are not require, but they include many useful features, by manupulate variables inside mb_query. The first query in the first station block will be executed, then first query in the secound station block will be executed and so on. When all the first queries has been executeded, then the secound query in all station block will be executed, starting with the first station block. Section 2, can still interfer this execution order. 

4. Each station block has there own internal loop of execution, when all queries has been executed the loop will start over again. If one SB has 5 queries while another has 15, then the first SB will have it's queries executed three times as fast as the second one.

5. By default only one query in each station block is executed for each loop. A bus is shared resource, and a device shouldn't be allowed to occupy the bus for long periods of time. However this cycle can be changed if some devices need more priority then otheres. By changing the variable «#sb.conf.exec_x_quries» in the station block, setting this variable to three for example, will let this one station block execute three queries for each loop, while other station blocks only execute one query on the same loop.

6. If a device encounter sveral timeout, then the communication-error-flag will be set for the station block, and queries will be executed less frequently. This make the trafic flow more effectively, by avoiding waiting for timeouts.

7. If more then one device have communication problems, then only one of them is allowed to make retry for each loop. The devices with error_comm-flag set will altnerative witch one of them will do a retry on the query loop.

8. Each SB has a disable flag, if it's set the queries inside the SB will be skipped, when it's the SB's turn to execute a query. The execution will continue in the next SB.

9. Each SB has a read_only flag. If this flag is set and the SB contains a write query. Then each time this write-query is going to be executed, the library whill skip the query, and let the following SB, execute a query instead. No query in the current SB will be executed at that loop. On the next loop the query after the write-query will be executed.

This are the main rule of execution, and all the rules apply at the same time. As mention on the top, the developer shouldn't need to worry about the order, she or he just need to know that they all will be executed, regardsless in which order.
