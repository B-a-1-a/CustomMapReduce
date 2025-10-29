## My Honors Project: MapReduce Plan

This is my implementation plan for the MapReduce project. It covers the `design.md` requirements and lays out a to-do list to get it done by the deadline.

### A Note on Go (vs. Rust)

> I originally considered using Rust for this. I was drawn to its performance and safety guarantees. However, looking at the project size and the Nov 9 deadline, I've decided to **switch to Go (Golang)**.
>
> I'm still a beginner with Rust, and the learning curve (especially the borrow checker) felt like a big risk for a project this complex.
>
> Go just makes more sense for this:
>
>   * **Concurrency:** **Goroutines** are perfect for this and way easier to manage than threads. I can spin up goroutines to manage workers, tasks, and heartbeats without a headache.
>   * **Garbage Collection:** I can focus on the system's logic (scheduling, data flow) instead of worrying about manual memory management.
>   * **Standard Library:** The `net/http` and `net/rpc` (or gRPC) packages are built for this kind of distributed system.
>   * **Simplicity:** Go compiles to a single static binary, which will make building the Docker images super simple and clean.

-----

### System Architecture & Design Decisions

Here are the main decisions for the `design.md`:

| Component | My Decision | Rationale |
| :--- | :--- | :--- |
| **Language** | **Go (Golang)** | As mentioned above. It's built for concurrent network services like this. |
| **Communication** | **gRPC** | I'll use gRPC for all communication. It's fast, and defining the services in a `.proto` file will keep things clean. <br> • **Client -\> Boss:** `SubmitJob(...)` <br> • **Boss -\> Worker:** `AssignTask(...)` <br> • **Worker -\> Boss:** `ReportHeartbeat(...)` |
| **Shared Storage** | **Docker Shared Volume** | Since it's a single VM, a standard Docker volume (e.g., `mapreduce-storage`) mounted to all containers is the simplest way. I'll just mount it at `/data` in the boss and all workers. |
| **Data Formats** | • **Input/Output:** Plain text (`.txt`). <br> • **Intermediate:** JSON Lines (e.g., `{"key": "k", "value": "v"}`). <br> • **Job File:** A compiled Go Plugin (`.so` file). |
| **Special Feature** | **Worker Failure Handling** | The Boss will expect periodic heartbeats from workers. If a worker misses too many, the Boss will assume it's dead, mark its in-progress tasks as "pending," and reschedule them on other, healthy workers. |

-----

### Deployment (Docker Compose)

The `docker-compose.yml` file will define the whole system:

  * **Services:**
      * `boss`: 1 container, running the main gRPC server that manages everything.
      * `worker`: 4 containers, running the worker gRPC server that just waits for tasks.
  * **CPU Limits:** I'll set `deploy.resources.limits.cpus: '1.0'` for the `worker` service so each is limited to one CPU, as required.
  * **Volumes:**
      * `mapreduce-storage`: A named volume that gets mounted to `/data` in the `boss` and all `worker` containers.

-----

### How It'll Work (Component Plan)

#### 1\. The Client (CLI)

This will be a simple Go binary (`./mr-client`). It won't do much work, just:

  * Take flags like `--input`, `--output`, `--jobfile`, `--maps`, `--reduces`.
  * Package this info into a gRPC `JobRequest`.
  * Send it to the Boss and report the response.

#### 2\. The Boss (Master)

This is the brains of the operation. It's a Go program that will:

  * Run a gRPC server to listen for `SubmitJob` calls from the client.
  * Keep a list of all workers and their status (e.g., `idle`, `busy`, `dead`).
  * Listen for heartbeats from workers.
  * When a job comes in:
    1.  Split the input files into M `MapTask`s.
    2.  Assign these tasks to `idle` workers.
    3.  When all `MapTask`s are done, assign R `ReduceTask`s to `idle` workers.
    4.  If a worker's heartbeat stops, re-queue any task it was working on.
    5.  When all `ReduceTask`s are done, mark the job as "complete."

#### 3\. The Worker

This is the "workhorse." It's a dumber Go program that will:

  * Run a gRPC server with one main method: `AssignTask`.
  * In a loop, send a `ReportHeartbeat` gRPC call to the Boss.
  * **When it gets a `MapTask`:**
    1.  Load the user's `.so` plugin file (path provided by the Boss).
    2.  Find the `Map` function inside it.
    3.  Read its assigned input file from `/data/...`.
    4.  Call the `Map` function for each record.
    5.  Partition the intermediate output into R files (one for each reducer) using `hash(key) % R`.
    6.  Write these intermediate files (e.g., `/data/intermediate/mr-map-X-reduce-Y.json`) to the shared volume.
    7.  Tell the Boss the task is done.
  * **When it gets a `ReduceTask`:**
    1.  Load the `.so` plugin and find the `Reduce` function.
    2.  Read all intermediate files for its partition (e.g., `mr-map-*-reduce-Y.json`).
    3.  Sort and group all the key-value pairs.
    4.  Call the `Reduce` function for each unique key.
    5.  Write the final output to `/data/output/` and tell the Boss it's done.

-----

### My To-Do List / Timeline

**Phase 1: Setup & API (By Oct 27)**

  * [ ] Create the GitHub/GitLab repo, add professor.
  * [ ] Define the gRPC `.proto` files (Job, Task, Status).
  * [ ] Create the initial `docker-compose.yml` with placeholder services.
  * [ ] Push this plan as `design.md`.

**Phase 2: Core Logic (Oct 28 - Nov 3)**

  * [ ] Build the Boss server: job/worker tracking, accept jobs.
  * [ ] Build the Worker server: accept tasks, send heartbeats.
  * [ ] Figure out the Go `plugin` package to load the `.so` files.
  * [ ] Get a "hello world" task to run from Boss -\> Worker.

**Phase 3: The "MapReduce" Part (Nov 3 - Nov 7)**

  * [ ] Implement the Map task logic (file splitting, partitioning).
  * [ ] Implement the Reduce task logic (the "shuffle" and sort).
  * [ ] Build the client CLI program.
  * [ ] Run a full Word Count job from start to finish.

**Phase 4: Feature & Polish (Nov 7 - Nov 9)**

  * [ ] Implement the "Special Feature": worker failure handling in the Boss.
  * [ ] Write tests\! Especially an integration test that kills a worker container mid-job.
  * [ ] Create the performance plots for the failure handling.
  * [ ] Finalize the `README.md` to be super clear.
  * [ ] **Push final code (Nov 9)**

**Phase 5: Demo (Nov 10 - 21)**

  * [ ] Schedule and pass the demo.

-----

### Testing & Evaluation Plan

  * **Unit Tests:** I'll use Go's built-in `testing` package for critical pure functions (like the reducer's sort/group logic and the `hash(key) % R` partitioner).
  * **Integration Test:** I'll write a `bash` script (or Go test) that:
    1.  Runs `docker-compose up`.
    2.  Copies the example job/data into the volume.
    3.  Runs `./mr-client submit ...`.
    4.  Waits for the job to finish and then `diff`s the output with a known-correct file.
  * **Failure Test (for the feature):** I'll add a step to the integration test that runs `docker kill <worker_container_name>` mid-job and confirms the job *still finishes* with the correct output.
  * **Performance Plot:** I'll make a bar chart showing **Job Time vs. \# of Failures**. The plot should demonstrate that while failures make the job slower, it *is* resilient and completes successfully, proving the feature works.

-----

### `README.md` Plan

My `README.md` will be a step-by-step guide.

1.  **Project:** What it is.
2.  **How to Build:** `docker-compose build`, `go build ./cmd/client`.
3.  **How to Run:** `docker-compose up -d`.
4.  **Full Example (Word Count):**
      * Here's the code for `jobs/wordcount.go`.
      * Here's how to compile it: `go build -buildmode=plugin ...`
      * Here's how to upload data.
      * Here's the *exact* `./mr-client submit ...` command to run it.
5.  **Performance (My Special Feature):** The plot showing failure handling in action.
