# Introduction
I do not possess the experience to completely deploy this within a very short weekend time-frame, and hence I have conducted research and written my approach given my available technical knowledge, interpretation, and experience. 

Steps to answer this question:

1. connect to Binance exchange API to stream orderbook data, WebSocket streams from endpoint “wss://stream.binance.com:9443”
2. Use asynchronous processes (e.g., asyncio in python) to handle the orderbook data and update the local orderbook
3. Design reconnection logic and error handling protocols to check whether there is an error in streaming data and ensure there is a local, secondary orderbook data backup
4. Identify the AWS region that is closest to the exchange’s matching engine, and test for latency through “pings” or AWS’s own latency engine

# Answers

## Q1: The orderbook must be stored in memory (at least 5 levels) & Q2: Justify your choice of the data structure used to store the orderbook data

Build a buy-order and sell-order dictionary of priority queues. The keys are the price levels, and the values will be a priority queue (either max-heap or min-heap) of orders at that price level. Each order has details such as quantity, or other information. The buy-order dictionary will be a max-heap priority queue, in the sense that data will be sorted over a tree in terms of maximum to minimum prices, whereas the sell-order dictionary will be a min-heap priority queue, meaning that order data is sorted over a tree in terms of minimum to maximum. Example would be, say, 5 levels of data for the buy orderbook, where my prices are [100, 90, 80, 70, 60] and order data associated with this will be retrieved. Furthermore, some relevant metrics that make it superior over other data structures:
1. Time Complexity
* Insertion in O(logN) time
* Retrieval of highest element (e.g., highest price) is O(1) time
* Extracting top 5 elements is O(5logN), if I’m re-updating / reheaping the node, otherwise it is O(n) for just finding the top 5 elements
2. Space Complexity
* Heap storage is O(N), where is the number of top levels maintained (at least 5)

The reason for using heaps is due to the nature of the orderbook. Speed is critical for quick insertion and retrieval for high frequency data and data analysis. O(logN) represents the best achievable time over O(1), so insertion is extremely efficient. Furthermore, retrieval of the best bid/ask prices comes at O(1) time, and if I wanted to find and maintain the top prices, I would be equally as quick with ~O(logN) time. In crypto, where my prices are in the decimals places, I could be expecting ~thousand prices every second / minute, so logN converges to a much lower value than its other data structure counterparts at O(N). 


## Q3: Updating the local version of the orderbook must be done asynchronously & Q4: The code design must be fail-safe such that the flow of the main application is not interrupted even if the exchange’s server goes down

To do this, I will implement an event loop using python’s asyncio library. Three components, (1) establish WebSocket connection, (2) message handling of the orderbook data, and (3) error handling. Updating the orderbook will be done in part (2), and a “local version” on our own server will maintain in case the connection breaks down to ensure that we can still briefly function without data incoming. If the exchange’s server goes down, there is a loop in place that retries to reconnect to the server, and in case there is an error for whatever other reason, there is a “try” and “except” block that should handle this by flagging this and stopping incoming trading processes. 

Furthermore, there should be another function in place such that if Binance exchange servers go down, we can still connect to another exchange server, such as coinbase, and combine past and future orderbooks to ensure trading continuity. 

## Q5: Find the AWS instance that has the lowest latency with the server where the exchange’s matching engine resides. Explain your approach

Assuming I do not know where the matching engine resides, I can interpolate. Go this website https://aws-latency-test.com/. It is AWS latency test that uses my IP Address to measure the median latency between each of Amazons’ data centers within a given region. Here, I am provided a round trip time (RTT), metric in milliseconds. Then, I use other tools, such as ping, traceroute, etc., to gather measurements of the amount of time it takes me to get information from Binance’s WebSocket. I would then compare it to my current list of median RTT, and the closest one to the Binance WebSocket will provide me with a rough estimate of where the exchange’s matching engine resides. 

Now that I roughly know where their servers are connected to, I can try to find out more specifically by trying out within a given sub-region. I can launch millions of artificial latency instances within this subregion to try to pinpoint where the exact location is (e.g., use traceroute, input location parameters, etc.) and without completely brute-forcing, I can interpolate between distances to try to approximate the latency I have with Binance. Ultimately, I can more or less use google satellites or find other cloud providers with servers close to the server location and therefore find out exactly where their matching engine resides.
