# docker-pm-resource-monitor
This Product Requirements Document (PRD) is designed to provide an LLM (like Gemini or Claude) with the specific architectural constraints and logic required to build your Notebook Controller & Monitor.
PRD: Single-Container Notebook Orchestrator & Monitor
1. Project Overview
The goal is to build a lightweight, single-container management system that can schedule, start, stop, and monitor Jupyter Notebooks using Papermill. The system must operate within a strict 16GB RAM limit and provide granular resource usage (CPU/RAM) for the container and individual notebooks.
2. Technical Stack
 * Backend: FastAPI (Python 3.10+)
 * Execution: Papermill (using asyncio.create_subprocess_exec)
 * Scheduling: APScheduler (BackgroundScheduler)
 * Monitoring: Hybrid approach using Cgroups v2 (container total) and psutil (per-process tree).
 * Frontend: Simple HTML/JavaScript UI (built-in FastAPI static files or Jinja2).
3. Functional Requirements
3.1 Notebook Management
 * Start Notebook: Trigger a Papermill execution of a specific .ipynb file.
 * Stop Notebook: Terminate the Papermill process and all its child kernel processes immediately.
 * Scheduling: Ability to set a Cron or Interval schedule for specific notebooks.
 * Persistence: Save executed notebooks with a timestamp (e.g., report_20260113.ipynb).
3.2 Resource Monitoring
 * Container Level: Read /sys/fs/cgroup/ to get total container RAM and CPU.
 * Process Level: Traverse the PID tree to sum memory/CPU for a Papermill process and its ipykernel children.
 * Real-time Updates: A WebSocket or polling endpoint to stream these stats to the UI.
3.3 Safety & Guardrails (The Watchdog)
 * Threshold: If Container RAM > 90% (approx. 14.5GB/16GB), the system must:
   * Identify the highest-consuming notebook process.
   * Kill that process to prevent an OOM (Out of Memory) crash of the entire container.
   * Log the event to the UI.
4. System Architecture
4.1 Process Mapping
The application must maintain an in-memory dictionary:
active_executions = {
    "notebook_id": {
        "process_handle": SubprocessObject,
        "start_time": datetime,
        "status": "running" | "failed" | "completed"
    }
}

5. Implementation Details for the LLM
5.1 Cgroup Monitoring Logic (v2)
The LLM should implement a ContainerMonitor class:
 * Memory: Read /sys/fs/cgroup/memory.current.
 * CPU: Read /sys/fs/cgroup/cpu.stat, specifically usage_usec. Calculate delta over time to get percentage.
5.2 Process Attribution Logic
The LLM should implement a function to aggregate child process memory:
def get_tree_memory(parent_pid):
    # Sum RSS of parent + all recursive children via psutil

5.3 API Endpoints
 * POST /notebooks/run/{name}: Starts execution.
 * DELETE /notebooks/stop/{name}: Kills the PID and children.
 * GET /stats: Returns JSON with {container_total: {}, individual_notebooks: []}.
 * POST /scheduler/add: Adds an APScheduler job.
6. UI Requirements (Simple Dashboard)
 * Header: Real-time progress bar of total 16GB Container RAM.
 * List View: Cards for each available notebook.
 * Actions: Start/Stop/Schedule buttons.
 * Live Metrics: Each "Running" card shows its own CPU% and RAM MB.
7. Instructions for Coding (Prompt for Gemini/Claude)
> "Act as an expert Python Backend Engineer. Use the provided PRD to create a single-file FastAPI application.
> Key constraints:
>  * Use asyncio.create_subprocess_exec for Papermill.
>  * Implement a 'Watchdog' background task that monitors cgroups and kills the heaviest process if RAM exceeds 15GB.
>  * Ensure the psutil logic correctly finds the ipykernel children of the Papermill process.
>  * Use APScheduler's BackgroundScheduler for the production scheduling requirement.
>  * Provide a simple internal index.html served by FastAPI to visualize these metrics."
> 
Here is the refined PRD. I have polished the technical specifications to ensure that an LLM like Gemini 1.5 Pro or Claude 3.5 Sonnet understands the specific nuances of process-tree monitoring and cgroup isolation.
PRD: Notebook Controller, Scheduler & Resource Monitor
1. Executive Summary
A lightweight, high-performance management layer for Jupyter Notebooks running inside a single Docker container. The system provides a FastAPI-based UI to trigger Papermill executions, schedule recurring jobs, and monitor resource usage to prevent container-wide crashes (OOM) within a 16GB limit.
2. Technical Constraints
 * Runtime: Python 3.10+ in a Linux Container.
 * Execution Engine: Papermill (executing as an asynchronous subprocess).
 * Scheduling: APScheduler (integrated into the FastAPI lifecycle).
 * Container Limit: 16GB RAM (Strict).
 * Monitoring Sources: * Cgroups v2: For total container health (authoritative).
   * Psutil: For individual notebook attribution (recursive PID tree).
3. System Architecture
3.1 Components
 * FastAPI Core: Manages the REST API and serves the monitoring dashboard.
 * Execution Manager: Tracks subprocess.Popen objects in a global state to allow for manual termination.
 * The Watchdog: A background loop that prevents "Out of Memory" (OOM) events by killing the largest consumer before the container crashes.
 * The Monitor: Aggregates CPU and RAM data from the OS.
4. Functional Requirements
4.1 Notebook Lifecycle
 * Asynchronous Start: Trigger papermill without blocking the API.
 * Forced Stop: Send SIGKILL to the entire process group (Papermill + Child Kernels).
 * Output Management: Store results as [notebook_name]_exec_[timestamp].ipynb.
4.2 Resource Monitoring (Reliability Specs)
 * Container RAM: Must read from /sys/fs/cgroup/memory.current.
 * Container CPU: Must calculate usage percentage by comparing two snapshots of /sys/fs/cgroup/cpu.stat (usage_usec).
 * Notebook Attribution: Must recursively sum the RSS (Resident Set Size) of the Papermill PID and its ipykernel children using psutil.
4.3 Scheduling
 * Support for Interval (e.g., every 10 minutes) and Cron (e.g., daily at 2 AM) scheduling.
 * Persistence of scheduled tasks within the application lifecycle.
5. Safety & Resiliency (The "Watchdog")
The application must implement a logic-based safety switch:
 * Polling Interval: 1 second.
 * Critical Threshold: 15GB (93% of 16GB).
 * Action: If Threshold is breached, identify the notebook_id with the highest get_tree_memory() and terminate it. Log "OOM Prevention: Terminated [Notebook Name]" to the UI.
6. Implementation Prompt for LLM (Copy & Paste)
> Role: Senior Python Systems Engineer.
> Task: Build a FastAPI application based on the provided PRD.
> Specific Coding Requirements:
>  * Subprocess Management: Use asyncio.create_subprocess_exec to run Papermill. Maintain a global dict active_tasks = {nb_name: process_object}.
>  * Cgroup Monitor: Implement a class CgroupMonitor that reads /sys/fs/cgroup/memory.current and calculates CPU % from /sys/fs/cgroup/cpu.stat.
>  * PID Tree Summation: Use psutil.Process(pid).children(recursive=True) to sum memory usage of both Papermill and its IPython kernels.
>  * The Watchdog: Create an asyncio background task that polls Cgroups. If memory > 15,000MB, kill the heaviest process in active_tasks.
>  * Scheduling: Integrate APScheduler (BackgroundScheduler).
>  * UI: Include a single-page HTML (using Tailwind CSS for styling) that shows:
>    * A global container RAM progress bar (0-16GB).
>    * A list of running notebooks with individual "Stop" buttons and real-time MB/CPU counters.
>    * A simple form to schedule a notebook.
> 
Since we are reading /sys/fs/cgroup, the container needs specific permissions and a specific base image to ensure psutil and papermill work correctly together.

