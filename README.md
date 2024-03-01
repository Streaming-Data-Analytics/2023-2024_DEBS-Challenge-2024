# DEBS 2024: Call for Grand Challenge Solutions

Optional project of the [Streaming Data Analytics](http://emanueledellavalle.org/teaching/streaming-data-analytics-2023-24/) course provided by [Politecnico di Milano](https://www11.ceda.polimi.it/schedaincarico/schedaincarico/controller/scheda_pubblica/SchedaPublic.do?&evn_default=evento&c_classe=811164&polij_device_category=DESKTOP&__pj0=0&__pj1=d563c55e73c3035baf5b0bab2dda086b).

Student: **[To be assigned]**


# Description

The DEBS Grand Challenge is a series of competitions that started in 2010, in which both participants from academia and industry compete with the goal of building faster and more scalable distributed and event-based systems that solve a practical problem. Every year, the DEBS Grand Challenge participants have a chance to explore a new data set and a new problem and can compare their results based on the common evaluation criteria. The winners of the challenge are announced during the conference where they are competing for a performance and an audience award. Apart from correctness and performance, submitted solutions are also assessed along a set of non-functional requirements.

### Topic: Telemetry data for hard drive failure prediction & predictive maintenance

The 2024 DEBS Grand Challenge focuses on real-time processing of real-world telemetry data provided by Backblaze ([https://www.backblaze.com/](https://www.backblaze.com/)).The data set used for the Grand Challenge contains fine-granular telemetry data about over 200k hard drives in data centers operated by Backblaze. The goal of the challenge is to continuously compute clusters of similar drives as new data is received. Further details on the data set provided, the queries, requirements and the overall submission process can be found here: [https://2024.debs.org/call-for-grand-challenge-solutions/](https://2024.debs.org/call-for-grand-challenge-solutions/)

## Project Goal
### Queries

This year’s DEBS Grand Challenge requires you to implement a count of the recent number of failures detected for each vault (group of storage servers) (Query Q1) and use this number to continuously compute a cluster of the drives (Query Q2).

#### Input

Input data consists of batches and each batch contains:

- SMART readings for a list of drives
- vault_ids: a list of vault identifiers of interest for this batch (used in Q1, see below)
- cluster_ids: a list of cluster identifiers of interest for this batch (used in Q2, see below)
- day_end: a flag that marks the end of one day of readings

### Query 1

- For every vault v, count the number of failures NF_v in a sliding window W
    - Size = 30 days
    - Slide = 1 day
- For a given day i, NF_v^i is the count of failures in the window that starts at i-31 (included) and ends at i-1 (included)
- The first window closes at day 0: you can assume 0 failures for every day i <= 0
- Each batch of input data will contain the identifiers of 5 vaults (field vault_ids): you are to return the current value of NF_v for those vaults as the query 1 response for the corresponding batch (Note: as NF_v^i uses the readings up to day i-1, the result can be sent for each batch without waiting for the end of the current day)

###  Query 2

- For each reading at day i regarding a drive d belonging to vault v, add NF_v^i to the reading
- Normalize data
    - Rescale the smart values within the provided ranges
    - The ranges can be downloaded from the [challenger platform](https://challenge2024.debs.org/)
- Compute dynamic K-means clustering
    - Assign incoming readings to the nearest centroid
    - At the end of one day, which is marked by a ‘day_end’ flag in the batch, update the centroids positions with the average coordinates of all the readings currently associated to that centroid
    - The initial coordinates of centroids are provided, and can be downloaded from the [challenger platform](https://challenge2024.debs.org/)
- Each batch of input data will contain a list of cluster identifiers (clusters are identified by a sequence number): you are to return the number of drives associated to each of those clusters

### Dataset Provided

The data provided for this DEBS Grand Challenge is based on S.M.A.R.T ([https://en.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology](https://en.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology)) data and additional attributes captured by Backblaze on a daily basis for six months (starting Q2 2023).

The data set contains events covering over 200k hard disks. Each data point resembles the S.M.A.R.T. status of a dedicated disk on a specific day.

The full data set will be used for benchmarking the submitted solutions. The smaller test data set will be provided upfront via our eval platform for testing purposes. The test data set contains event notifications representing six months of data about drives only from a single manufacturer. Consequently, participants must not pay attention to filtering out event notifications that do not contain attributes relevant to the Grand Challenge.

The events are provided through a GRPC based API (see the protobuf definition below). The input data is provided in numbered Batches of DriveState. Each DriveState is a reading identified by a timestamp and the serial number of the drive. The reading contains the model of the drive, the id of the vault where it is located and a boolean marking whether it has failed. The DriveState contains a list of readings for the smart attributes of the drive at that specific timestamp. It is provided as a list of int64 raw readings, the values map to the smart attributes in the following order: s1,s2,s3,s4,s5,s7,s8,s9,s10,s12,s173,s174,s183,s187,s188,s189,s190,s191,s192,s193,s194,s195,s196,s197,s198,s199,s200,s220,s222,s223,s226,s240,s241,s242


```json
syntax = "proto3";

    import "google/protobuf/empty.proto";
    import "google/protobuf/timestamp.proto";
    
    option java_multiple_files = true;
    option java_package = "org.debs.gc2023.bandency";
    
    package Challenger;
    
    message DriveState {
        google.protobuf.Timestamp date = 1;
        string serial_number = 2;
        string model = 3;
        bool failure = 4;
        int32 vault_id = 5;
        // SMART raw readings in the following order:
        // s1,s2,s3,s4,s5,s7,s8,s9,s10,
        // s12,s173,s174,s183,s187,s188,
        // s189,s190,s191,s192,s193,s194,
        // s195,s196,s197,s198,s199,s200,
        // s220,s222,s223,s226,s240,s241,
        // s242      
        repeated int64 readings = 6;
    }
    
    message Batch {
        int64 seq_id = 1;
        bool last = 2;
        bool day_end = 3;
        repeated int32 vault_ids = 4;
        repeated int32 cluster_ids = 5;
        repeated DriveState states = 6;
    }
    
    message Benchmark {
        int64 id = 1;
    }
    
    message VaultFailures {
        int32 vault_id = 1;
        int32 failures = 2;
    }
    
    message ResultQ1 {
        int64 benchmark_id = 1;
        int64 batch_seq_id = 2;
    
        repeated VaultFailures entries = 3;
    }
    
    message ClusterInfo {
        int32 cluster_id = 1;
        int32 size = 2;
    }
    
    message ResultQ2 {
        int64 benchmark_id = 1;
        int64 batch_seq_id = 2;
    
        repeated ClusterInfo entries = 3;
    }
    
    enum Query {
        Q1 = 0;
        Q2 = 1;
    }
    
    message BenchmarkConfiguration {
        string token = 1; // Token from the webapp for authentication
        string benchmark_name = 2; // chosen by the team, listed in the results
        string benchmark_type = 3; // benchmark type, e.g., test
        repeated Query queries = 4; // Specify which queries to run
    }
    
    service Challenger {
    
        //Create a new Benchmark based on the configuration
        rpc createNewBenchmark(BenchmarkConfiguration) returns (Benchmark);
    
        //This marks the starting point of the throughput measurements
        rpc startBenchmark(Benchmark) returns (google.protobuf.Empty);
    
        //get the next Batch
        rpc nextBatch(Benchmark) returns (Batch);
    
        //post the result
        rpc resultQ1(ResultQ1) returns (google.protobuf.Empty);
        rpc resultQ2(ResultQ2) returns (google.protobuf.Empty);
        
        //This marks the end of the throughput measurements
        rpc endBenchmark(Benchmark) returns (google.protobuf.Empty);
    }
```                

HTTP Accessibility API Example

```json
POST /create HTTP/1.1
    Content-Length: 96
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "token": "pepega",
        "benchmark_name": "asd",
        "benchmark_type": "test",
        "queries": [0]
    }
    
    
    POST /start HTTP/1.1
    Content-Length: 18
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "id": 734652
    }
    
    
    POST /end HTTP/1.1
    Content-Length: 18
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "id": 734652
    }
    
    
    POST /next_batch HTTP/1.1
    Content-Length: 18
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "id": 734652
    }
    
    
    POST /result_q1 HTTP/1.1
    Content-Length: 136
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "benchmark_id": 734652,
        "batch_seq_id": 123,
        "entries": [
        {
            "model": "AB12",
            "intervals": ["a", "b"]
        }
        ]
    }
    
    
    POST /result_q2 HTTP/1.1
    Content-Length: 104
    Content-Type: application/json
    Host: 127.0.0.1:3000
    User-Agent: HTTPie
    
    
    {
        "benchmark_id": 734652,
        "batch_seq_id": 123,
        "centroids_out": [1, 2],
        "centroids_in": [3, 4]
    }
                                                                  

```

# [Optional] Participation

Participation in the DEBS 2024 Grand Challenge consists of three steps: (1) registration, (2) iterative solution submission, and (3) paper submission.

The first step is to pre-register your submission by registering your abstract at Easychair [https://easychair.org/my/conference?conf=debs24](https://easychair.org/my/conference?conf=debs24) in the "Grand Challenge Track'' and send an email to one of the Grand Challenge Co-Chairs at [debs24gc@gmail.com](mailto:debs24gc@gmail.com) (see [https://2024.debs.org/organizing-committee/](https://2024.debs.org/organizing-committee/) or the eval platform landing page). Solutions to the challenge, once developed, must be submitted to the evaluation platform ( [https://challenge2024.debs.org/](https://challenge2024.debs.org/)) to get benchmarked in the challenge. The evaluation platform provides detailed feedback on performance and allows the solution to be updated in an iterative process. A solution can be continuously improved until the challenge closing date. Evaluation results of the last submitted solution will be used for the final performance ranking. The last step is to upload a short paper (minimum 2 pages, maximum 6 pages) describing the final solution via the central conference management tool Easychair. The DEBS Grand Challenge Committee will review all papers to assess the merit and originality of submitted solutions. All solutions of sufficient quality will be presented during the poster session at the DEBS 2024 conference.


### Awards and Selection Process

Participants of the challenge compete for two awards: (1) the performance award and (2) the audience award. The winner of the performance award will be determined through the automated evaluation platform Challenger (see above), according to the evaluation criteria specified below. Evaluation criteria measure the speed and correctness of submitted solutions. The audience award winner will be determined amongst the finalists who present in the Grand Challenge session of the DEBS conference. In this session, the audience will be asked to vote for the solution with the most exciting concepts. The solution with the highest number of votes wins. The audience award intends to highlight the qualities of the solutions that are not tied to performance. Specifically, the audience and challenge participants are encouraged to consider the functional correctness and the practicability of the solution.

Regarding the practicability of a solution, we want to encourage participants to push beyond solving a problem correctly only in the functional sense. Instead, the aim is to provide a reusable and extensible solution that is of value beyond this year’s Grand Challenge alone. In particular, we will reward those solutions that adhere to a list of non-functional requirements. These requirements are driven by industry use cases and scenarios so that we make sure solutions are applicable in practical settings.

Thus, every submission will be evaluated for how it addresses our non-functional requirements in its design and implementation while assuming that all submitted solutions strive for maximum performance. In this regard, we distinguish between hard and soft non-functional requirements.

- **Hard** criteria are must-haves that have to be addressed in the design and must be implemented; while
- **Soft** criteria do not necessarily have to be implemented but must be covered in the submission's description

Examples for hard criteria are: Configurability, Scalability (horizontal scalability is preferred), operational reliability/resilience, accessibility of the solution's source code, integration with standard (tools/protocols), documentation.

Examples for soft criteria are: Security measures implemented/addressed, deployment support, portability/maintainability, and support of special hardware (e.g., FPGAs, GPUs, SDNs,...).

To achieve all non-functional requirements, we want to discourage participants from building their solution from scratch and consider the widely-used industry-strength open-source platforms like those curated by recognised open-source foundations ([https://opensource.com/resources/organizations](https://opensource.com/resources/organizations)). We promote these platforms not only because they already have a diverse user base and ecosystem but also because most of them already solve non-functional requirements that are paramount for use in practice.

There are two ways for teams to become finalists and get a presentation slot in the Grand Challenge session during the DEBS Conference: (1) up to two teams with the best performance (according to the final evaluation) will be nominated; (2) the Grand Challenge Program Committee will review submitted papers for each solution and nominate up to two teams with the most novel concepts. All submissions of sufficient quality that do not make it to the finals will get a chance to be presented at the DEBS conference as posters. The quality of the submissions will be determined based on the review process performed by the DEBS Grand Challenge Program Committee. In the review process, the non-functional requirements will receive special attention. Furthermore, participants are committed to presenting details on which non-functional requirements are implemented and to what level of sophistication in their presentation slot. The audience will be encouraged to pay special attention to these details when voting for the best solution.

### Important Dates

- Test platform opens: **January, 2024**
- Evaluation platform opens: **March, 2024**
- Deadline for uploading final solution to the evaluation platform: **May 12th, 2024**
- Evaluation of the solution: **May 19th, 2024**
- Deadline for short paper submission: **June 2nd, 2024**
- Notification of acceptance: **June 9th, 2024**
- Camera ready submission: **June 16th, 2024**
- Conference **June 25th – 28th June, 2024**

### Grand Challenge Co-Chairs

- Alessandro Margara, Politecnico di Milano, Italy
- Sebastian Frischbier, Allianz Global Investors
- Jawad Tahir, TU Munich
- Christoph Doblander, TU Munich
- Luca De Martini, Politecnico di Milano, Italy

## Note for Students

* Clone the created repository offline;
* Add your name and surname into the Readme file;
* Make any changes to your repository, according to the specific assignment;
* Add a `requirement.txt` file for code reproducibility and instructions on how to replicate the results;
* Commit your changes to your local repository;
* Push your changes to your online repository.
