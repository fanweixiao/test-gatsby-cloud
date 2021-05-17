---
title: "Implement DEBS 2014 Grand Challenge with YoMo, an Open Source Real-time Stream Processing Framework"
date: 2021-05-13
slug: "/implement-debs-2014-grand-challenge-with-yo-mo-an-open-source-real-time-stream-processing-framework"
canonicalUrl: "https://blog.yomo.run"
description: "DEBS.org is a global Distributed and Event-Based Systems (DEBS) conferences and workshops. Since 2011, the DEBS conference has provided a yearly Grand Challenge to the event processing community. The goal of the DEBS Grand Challenge is to provide a common ground and evaluation criteria for a competition aimed at both research and industrial event-based systems."
author: Ivy Guo (Intern)
tag: use-case
---

## Introduction

[DEBS](https://dl.acm.org/conference/debs) or the **ACM International Conference on Distributed Event Based Systems** aims to "provide a forum dedicated to the dissemination of original research, the discussion of practical insights, and the reporting of experiences relevant to event based computing that were previously scattered across several scientific and professional communities" ([reference](http://www.wikicfp.com/cfp/servlet/event.showcfp?eventid=34432)).

[The DEBS 2014 Grand Challenge - Smart Grid](https://debs.org/grand-challenges/2014/) is the 8th ACM International Conference on Distributed Event Based Systems, focused on two problems which are relevant to the industry: *real-time load prediction* and *anomaly detection*. The data for the challenge was collected from a number of smart-home installations deployed in Germany.

In traditional processing, data is often stored in the database at first, and then processed to obtain useful information at a secondary stage. The traditional architecture:

![yomo debs2014 storm arch](/debs2014-storm.png)

However, with stream processing, we're swiftly able to run real-time analytics on incoming data streams. By utilizing Yomo, an open-source serverless stream processing framework, we can address DEBS' challenge in a real-time fashion:

<img alt="yomo debs2014 new arch" src="/debs2014-yomo.png" width="100%" />

Firstly, we will take a look at the type of data we're dealing with. Next, we will introduce two queries that were originally proposed by ACM DEBS. Lastly, we install [YoMo](http://github.com/yomorun/yomo) (an open-source framework for real-time stream processing) to implement both queries as described.

## Data

From Jerzak and Ziekow (2014):

> For the DEBS 2014 Grand Challenge we assume a hierarchical structure with a house, identified by a unique house id, being the topmost entity. Every house contains one or more households, identified by a unique household id (within a house). Every household contains one or more smart plugs, each identified by a unique plug id (within a household). Every smart plug contains exactly two sensors:
> (1) <u>load</u> sensor measuring current load with Watt as unit (2) <u>work</u> sensor measuring total accumulated work since the start (or reset) of the sensor with kWh as unit.

The input stream is defined as follows:

- `id` – a unique identifier of the measurement [32 bit unsigned int]
- `timestamp` – timestamp of measurement [32 bit unsigned int]
- `value` – the measurement [32 bit float]
- `property` – type of the measurement: 0 for work or 1 for load [boolean]
- `plug_id` – a unique identifier (within a household) of the smart plug [32 bit unsigned int]
- `household_id` – a unique identifier of a household (within a house) where the plug is located [32 bit unsigned int]
- `house_id` – a unique identifier of a house where the household with the plug is located [32 bit unsigned int]

The complete data file is available under [this link](http://www.doc.ic.ac.uk/~mweidlic/sorted.csv.gz). For demonstration purposes, we will generate mock data using this file. In real life, we should be dealing with direct sensor data.

## Queries

1. Load Prediction

    Let's divide the whole period from `t_{start} = 1377986401` to `t_{end} = 1380578399` covered by the dataset into `N` equal slices of `|s|` seconds and call them `s_0`, `s_1`, `s_2`, etc.

    The average load for slice `s_{i + 2}` is given by

    ```text
    L(s_{i + 2}) = ( avgLoad(s_i) + median({ avgLoad(s_j) }) ) / 2
    ```

    `s_j` is a nonempty set defined by `s_(i + 2 – n * k)`, where `k` is the number of slices within a 24-hour period and `n` is a natural number between `1` and `floor((i + 2) / k)`.

2. Outliers

    For this query, we will compute the percentage of plugs which have a median load during the last hour greater than the median load of all plugs.

## YoMo – what is it and why do we use it?

[YoMo](https://github.com/yomorun/yomo) is an open-source serverless streaming framework for building low-latency edge computing applications. Built atop QUIC transport protocol and functional reactive programming interface, it makes real-time data processing reliable, secure, and easy.

## Getting Started

1. Follow the instructions [here](https://yomo.run/?utm_source=blog&utm_campaign=cc) to install YoMo. Assuming that `$GOPATH` has been set on your device, you should be able to see a directory with the name `$GOPATH/src/github.com/yomorun/yomo`.

2. Run the following commands to create a new project. Don't forget to replace `${YOUR_GITHUB_USERNAME}` with your actual GitHub username!

```shell
$ mkdir -p $GOPATH/src/github.com/${YOUR_GITHUB_USERNAME} && cd $_
$ yomo init debs-flow
```

Now, `cd` into `$GOPATH/src/github.com/${YOUR_GITHUB_USERNAME}`. It should have the following structure.

```shell
.
└── debs-flow
		├── app.go
		├── go.mod
		├── go.sum
		└── sl.so
```

We will be focusing on the `Handler` function in `app.go`, which defines how we want the input stream to be processed.

3. Git clone the [`yomo-source-example` repository](https://github.com/yomorun/yomo-source-example). For clarity, we will call it `debs-source`.

```shell
$ git clone git@github.com:yomorun/yomo-source-example.git
```

4. Create a file named `workflow.yaml` with the following content. Place it under `debs-source`.

```yaml
name: service
host: localhost
port: 9000
flows:
- name: debs 2014
		host: localhost
		port: 4242
sinks:
- name: mock db
		host: localhost
		port: 4243
```

Make sure the `$GOPATH/src/github.com/${YOUR_GITHUB_USERNAME}` directory contains the following files.

```shell
.
├── debs-flow
│   ├── app.go
│   ├── go.mod
│   ├── go.sum
│   └── sl.so
└── debs-source
		├── go.mod
		├── go.sum
		├── main.go
		└── workflow.yaml
```

## Algorithm Implementation

By default, the `Handler` function in `debs-flow/app.go` should look as follows.

```go
func Handler(rxstream rx.RxStream) rx.RxStream {
	stream := rxstream.
		Subscribe(0x10).
		OnObserve(decoder).
		Map(printer).
		Encode(0x11)

	return stream
}
```

- `Subscribe(0x10)`: Subscribe to the input stream. `0x10` is the key. It is defined in `debs-source/main.go`.
- `OnObserve(decoder)`: Decode `[]byte` to `interface{}`. Empty interfaces are often used by code that handles values of unknown type.
- `Map(printer)`: Print out the data.

For query #1, we need to define two functions in addition to what we discussed above. We will call them `average` and `predict`.

```go
func Handler(rxstream rx.RxStream) rx.RxStream {
	stream := rxstream.
		Subscribe(0x10).
		OnObserve(decoder).
		Map(printer).
		BufferWithTime(ss * 1e3). // ss stands for slice size
		Map(average).
		Map(predict).
		Encode(0x11)

	return stream
}
```

```go
// compute the average load for each plug and save the values to db, which is a global variable in this example
func average(_ context.Context, i interface{}) (interface{}, error) {
	// convert interface{} to []interface{}
	lst, ok := i.([]interface{})
	if !ok {
		err := fmt.Sprintf("expected type '[]interface{}', got '%v' instead",
			reflect.TypeOf(i))
		fmt.Printf("[average] %v\n", err)
		return nil, fmt.Errorf(err)
	}

	// plug # -> value
	total := make(map[string]float32)
	count := make(map[string]float32)

	for _, elem := range lst {
		// convert interface{} to measurement
		x, ok := elem.(Measurement)
		if !ok {
			err := fmt.Sprintf("expected type 'measurement', got '%v' instead",
				reflect.TypeOf(elem))
			fmt.Printf("[average] %v\n", err)
			return nil, fmt.Errorf(err)
		}

		if x.Property { // load
			plug := x.toString()
			total[plug] += x.Value
			count[plug] += 1.0
		}
	}

	// save to db
	fmt.Println("*** average ***")
	for plug, v := range total {
		avg := v / count[plug]
		fmt.Printf("[s_%v] %v %v\n", idx, plug, avg)

		_, ok := db[plug]
		if !ok {
			db[plug] = make(map[uint32]float32)
		}
		db[plug][idx] = avg
	}
	fmt.Println("***************")
	return i, nil
}

// make predictions based on what we have in db
func predict(_ context.Context, i interface{}) (interface{}, error) {
	k := t / ss

	fmt.Println("*** predict ***")
	l := (idx + 2) / k
	if l == 0 {
		fmt.Println("not enough data")
	} else {
		// possible values for j
		lst := make([]uint32, l)
		for m := range lst {
			n := uint32(m + 1)
			j := idx + 2 - n*k
			lst[m] = j
		}

		for plug := range db {
			// average load for s_j
			data := make([]float32, l)
			for m, j := range lst {
				data[m] = db[plug][j]
			}
			pred := (db[plug][idx] + median(data)) / 2
			fmt.Printf("[s_%v] %v %v\n", idx+2, plug, pred)
		}
	}
	fmt.Println("***************")

	idx += 1 // slice #
	return 0.0, nil
}
```

For query #2, we will define a function called `outliers`.

```go
func Handler(rxstream rx.RxStream) rx.RxStream {
	stream := rxstream.
		Subscribe(0x10).
		OnObserve(decoder).
		Map(printer).
		BufferWithTime(ss * 1e3).
		Map(outliers).
		Encode(0x11)

	return stream
}
```

```go
// which plugs have a median load greater than the median load of all plugs?
func outliers(_ context.Context, i interface{}) (interface{}, error) {
	// convert interface{} to []interface{}
	lst, ok := i.([]interface{})
	if !ok {
		err := fmt.Sprintf("expected type '[]interface{}', got '%v' instead",
			reflect.TypeOf(i))
		fmt.Printf("[outliers] %v\n", err)
		return nil, fmt.Errorf(err)
	}

	all := make([]float32, 0, len(lst))
	indiv := make(map[string][]float32) // plug # -> values

	for _, elem := range lst {
		// convert interface{} to measurement
		x, ok := elem.(Measurement)
		if !ok {
			err := fmt.Sprintf("expected type 'measurement', got '%v' instead",
				reflect.TypeOf(elem))
			fmt.Printf("[outliers] %v\n", err)
			return nil, fmt.Errorf(err)
		}

		if x.Property { // load
			all = append(all, x.Value)

			plug := x.toString()
			indiv[plug] = append(indiv[plug], x.Value)
		}
	}

	v := median(all)
	fmt.Printf("all plugs: %v\n", v)

	fmt.Println("*** outliers ***")
	for plug, vs := range indiv {
		m := median(vs)
		if m > v {
			fmt.Printf("[w_%v] %v %v\n", idx, plug, m)
		}
	}
	fmt.Println("****************")

	idx += 1
	return 0.0, nil
}
```

Now to run the code, we need to:

1. `cd` into `debs-flow` and type

    ```shell
    $ yomo run app.go
    ```

2. `cd` into `debs-source` and type

    ```shell
    $ yomo wf run workflow.yaml
    $ PORT=9000 go run main.go
    ```

You might want to try a different set of hyperparameters.

## Results

For query #1, you should see something similar to the following.

```text
...
[1620461050] 9.910085 0-1-2 load
[1620461050] 8.087268 0-1-2 work
[1620461050] 13.468374 3-1-2 load
[1620461050] 7.742124 3-1-2 work
[1620461051] 13.738256 0-1-2 load
[1620461051] 16.59261 0-1-2 work
[1620461051] 12.84997 3-1-2 load
[1620461051] 10.838872 3-1-2 work
*** average ***
[s_18] 0-1-2 11.824171
[s_18] 3-1-2 13.159172
***************
*** predict ***
[s_20] 0-1-2 10.375039
[s_20] 3-1-2 12.0406475
***************
...
```

For query #2, something like this:

```text
...
[1620461271] 6.921172 0-1-2 load
[1620461271] 1.8683584 0-1-2 work
[1620461271] 17.251171 3-1-2 load
[1620461271] 9.761936 3-1-2 work
[1620461272] 10.758014 0-1-2 load
[1620461272] 18.668419 0-1-2 work
[1620461272] 5.806175 3-1-2 load
[1620461272] 1.8562717 3-1-2 work
[1620461273] 0.11624338 0-1-2 load
[1620461273] 5.579194 0-1-2 work
[1620461273] 17.249205 3-1-2 load
[1620461273] 4.9580107 3-1-2 work
[1620461274] 8.087428 0-1-2 load
[1620461274] 7.49426 0-1-2 work
[1620461274] 4.6709924 3-1-2 load
[1620461274] 1.793222 3-1-2 work
[1620461275] 4.8114495 0-1-2 load
[1620461275] 1.9070174 0-1-2 work
[1620461275] 19.199306 3-1-2 load
[1620461275] 9.054778 3-1-2 work
all plugs: 7.5043
*** outliers ***
[s_5] 3-1-2 17.249205
****************
...
```

## About Author

Ivy Guo is a Computer Science student at the University of Washington. If you have any questions, please email Ivy at zhifeig@cs.washington.edu

## Further Reading

- Rohit Gupta, Rinku Shah, and Apurva Mhetre. 2014. In-Memory, High Speed Stream Processing. In *Proceedings of the 8th ACM International Conference on Distributed Event-Based Systems* (*DEBS '14*). Association for Computing Machinery, New York, NY, USA, 306–309. DOI: https://doi.org/10.1145/2611286.2611332.

- Abhinav Sunderrajan, Heiko Aydt, and Alois Knoll. 2014. Real-Time Load Prediction and Outliers Detection using STORM. DEBS 2014 - *Proceedings of the 8th ACM International Conference on Distributed Event-Based Systems*. 10.1145/2611286.2611327. [ResearchGate](https://www.researchgate.net/publication/262419725_DEBS_Grand_Challenge_Real_time_Load_Prediction_and_Outliers_Detection_using_STORM)

- ACM DEBS Grand Challenge 2014 implementation using Apache Flink: [Github](https://github.com/koldbyte/smartgrid)

- DEBS '14: Proceedings of the 8th ACM International Conference on Distributed Event-Based Systems [all research papers](https://dl.acm.org/doi/proceedings/10.1145/2611286)

## References

- Zbigniew Jerzak and Holger Ziekow. 2014. The DEBS 2014 grand challenge. In *Proceedings of the 8th ACM International Conference on Distributed Event-Based Systems* (*DEBS '14*). Association for Computing Machinery, New York, NY, USA, 266–269. DOI: https://doi.org/10.1145/2611286.2611333.

## More about YoMo

https://github.com/yomorun/yomo