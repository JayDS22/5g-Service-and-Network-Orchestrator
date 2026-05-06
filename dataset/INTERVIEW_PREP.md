# Interview Prep — Service-Aware Real-Time Slicing for Virtualized beyond 5G Networks

## One-line pitch
A cloud-native 5G network where an AI sidecar predicts per-application traffic in real time and reconfigures RAN radio-resource slices every second to protect Quality of Experience for latency-sensitive apps.

---

## STAR description

### Situation
Modern 5G networks carry a heterogeneous mix of traffic — interactive video (WebRTC), signaling/voice (SIP), and bulk web traffic — over the same shared radio spectrum. Operators allocate spectrum to **RAN slices**, but static slice percentages over-provision some apps and starve others, while purely reactive slicing reacts only after QoE has already degraded. A telecom operator needs a way to **anticipate** the next few seconds of traffic per user and per application and to reshape the radio allocation **before** congestion happens, all running on commodity Kubernetes infrastructure rather than vendor-locked hardware.

### Task
Design and deploy a service-aware, AI-driven orchestrator that:
1. Runs the entire 5G core and RAN cloud-natively on Kubernetes using OpenAirInterface (OAI).
2. Observes live user-plane traffic and radio metrics in real time without disrupting the data path.
3. Forecasts each user's traffic mix for the next time window using a deep-learning model.
4. Translates the forecast into per-UE slice percentages and pushes them to the RAN controller (FlexRAN) within sub-second loops.
5. Supports continuous retraining via an MLOps pipeline so the model adapts as traffic patterns drift.

### Action
- **Cloud-native 5G deployment**: Brought up the full OAI stack on a 3-node Kubernetes cluster (`master`, `cloud-worker`, `antenna-worker` with USRP B210 SDR) — HSS, MME, SPGW-C, SPGW-U, CU, DU — alongside Cassandra (subscriber DB), MySQL (telemetry), and the FlexRAN controller. Used Multus CNI for multi-interface pods, NFS provisioner for shared model storage, and pinned radio-touching pods to the SDR node via `nodeName`.
- **Live telemetry plane**: Implemented a `pyshark`-based packet sniffer as a **sidecar container** inside the SPGW-U pod, tapping interface `net3` so it sees every UE↔application packet with zero impact on the user plane. Bucketed packets into **1-second mini-windows** keyed by (UE, application).
- **Feature engineering (15 features × 30 time-steps)**: Per mini-window, computed 9 throughput series (3 UEs × 3 apps: WebRTC / SIPp / web-server), 3 per-UE jitter values from packet inter-arrival times, and 3 per-UE **CQI** values pulled from FlexRAN's REST API. Stacked the last 30 mini-windows into a `(1, 30, 15)` tensor and `MinMaxScaler`-normalized it with a scaler fitted at training time.
- **Model selection**: Trained and benchmarked **eight architectures** — LSTM, GRU, Bi-LSTM, Bi-GRU, CNN-LSTM, CNN-GRU, CNN-FNN, FNN — on the same dataset. **CNN-LSTM** won (CNN layers extract local correlations across the 15 features; LSTM captures temporal dependencies across the 30-step window). Loaded the winning `.hdf5` weights and joblib scaler from an NFS-mounted volume at `/mnt/`.
- **Slicing decision policy**: Mapped each model prediction (next-window throughput / jitter / CQI per UE) to discrete priority levels via threshold filters, then computed the slice percentage as a weighted sum `4·app + 4·throughput + 4·jitter + 4·CQI + 8`. Compared against the current slices and POSTed updates to FlexRAN's `/slice/enb/-1` endpoint only when the decision changed, avoiding unnecessary churn. Initial UE-to-slice association is set once at boot using each UE's **RNTI**.
- **MLOps**: Authored a **Kubeflow Pipeline** in three components — `step1` (extract sliding-window X/y from raw CSVs), `step2` (scale + serialize to JSON), `step3` (train LSTM, save `.hdf5` back to NFS) — compiled with `kfp.compiler` into a YAML manifest uploaded to the Kubeflow service for repeatable, scheduled retraining.
- **End-to-end validation**: Drove three real LTE-dongle UEs running per-scenario traffic generators (`scenarios/Traffic/scenarioN/ueN.sh`) — Chromium WebRTC sessions, SIPp call flows, and curl-driven web traffic — and compared QoE (jitter, throughput, MAE of predictions) before vs after slicing in `gnu/QoE/` plots.

### Result
- Achieved **per-second proactive RAN re-slicing** end-to-end, replacing static or reactive policies.
- Demonstrated measurable QoE improvement (lower jitter, higher sustained throughput for latency-sensitive WebRTC) on real radio hardware, not just simulation.
- Shipped a fully **reproducible cloud-native deployment**: a single `bash deploy-all` brings up core + RAN + apps + AI sidecar; `destroy-all` tears it down.
- Work was published in **Computer Networks (Elsevier), Vol. 247, 2024** — *"Service-aware real-time slicing for virtualized beyond 5G networks"* (DOI: 10.1016/j.comnet.2024.110445).

---

## Cross-question concept cheat-sheet

### 5G core / RAN domain

- **RAN slicing** — Partitioning the radio interface into virtual slices, each with its own share of time/frequency resources, so different services (eMBB, URLLC, mMTC) get isolated QoS guarantees on the same physical infrastructure.
- **OpenAirInterface (OAI)** — Open-source software implementation of the 4G/5G stack (eNB/gNB, EPC, 5GC). Used here so the entire RAN and core can run as containers on commodity x86 + USRP.
- **FlexRAN** — A real-time RAN controller / SDN-style agent for OAI that exposes a REST API (`/stats`, `/slice/enb/-1`, `/ue_slice_assoc/enb/-1`) to read radio metrics and reconfigure scheduling and slicing on the fly.
- **CU / DU split** — 3GPP functional split where the gNB is decomposed into a Centralized Unit (higher-layer protocols, can be virtualized in cloud) and a Distributed Unit (lower-layer, near the radio). Allows centralized orchestration with low-latency radio processing.
- **HSS / MME / SPGW-C / SPGW-U** — 4G EPC functions: Home Subscriber Server (auth/profile), Mobility Management Entity (control), Serving/PDN GW Control plane, Serving/PDN GW User plane. The user-plane traffic flows through SPGW-U, which is exactly why the sniffer sidecar lives there.
- **CQI (Channel Quality Indicator)** — A 0–15 integer the UE reports to the base station describing radio link quality; the scheduler uses it to pick modulation/coding. Lower CQI ⇒ worse channel ⇒ that UE needs more resource share to maintain throughput.
- **RNTI (Radio Network Temporary Identifier)** — A short ID the cell assigns to each connected UE used to address it on the air interface. Used in this project to bind UEs to slice IDs.
- **IMSI / APN / PLMN** — IMSI uniquely identifies a SIM globally; APN is the gateway name the UE attaches to (`apn.oai.svc.cluster.local` here, resolved via Kubernetes DNS); PLMN is the operator code (MCC+MNC, `46099` in the testbed).
- **USRP B210 / SDR** — Software-Defined Radio from Ettus; lets a generic PC act as a base station radio front-end driven by OAI software.
- **Jitter** — Variance in packet inter-arrival time; matters for real-time apps (voice, video) more than for bulk transfer. Computed here as the mean delta between consecutive packet timestamps per UE.
- **QoS vs QoE** — QoS is what the network measures (throughput, latency, loss); QoE is what the user perceives (smooth video, no echo). Service-aware slicing optimizes for QoE by giving each app the QoS profile it actually needs.

### Machine learning

- **CNN-LSTM hybrid** — 1-D CNN layers act as learned feature extractors over the multivariate input (capture short-range patterns across the 15 columns), then LSTM layers model the temporal evolution across the 30-step window. Often outperforms pure LSTM on multivariate sensor-style data.
- **LSTM / GRU** — Recurrent units with gating that mitigate the vanishing-gradient problem; LSTM has separate forget/input/output gates and a cell state, GRU merges them into update/reset gates and is lighter and faster with comparable accuracy.
- **Bidirectional RNN** — Runs one RNN forward and one backward over the sequence and concatenates the outputs; useful when future context helps (offline analysis) but adds latency for streaming inference.
- **Sliding window for time-series forecasting** — Convert a long time series into supervised `(X, y)` pairs by sliding a fixed-length window: `X = series[i : i+N]`, `y = mean(series[i+N : i+N+M])`. Here N=30 mini-windows, M is the prediction horizon.
- **MinMaxScaler** — Linearly rescales each feature into `[0, 1]` using the training min/max; the same scaler must be persisted (`joblib`) and reused at inference, otherwise the model sees out-of-distribution inputs.
- **Inverse transform** — Applied to the model's prediction to map the `[0, 1]` output back to original units (bytes, ms, CQI levels) before threshold-based decisions.
- **MAE (Mean Absolute Error)** — Average absolute difference between predicted and true values; preferred over MSE when outliers shouldn't dominate the loss signal. Used in the paper's evaluation plots (`gnu/mae/`).
- **Online / continuous training** — Periodically retraining the model on freshly captured traffic so it adapts to drift; in this project, automated via the Kubeflow pipeline and shown in `gnu/online_training/`.
- **Data augmentation for scenarios** — `Augmentation/Scenario Augmentation.ipynb` synthesizes additional traffic scenarios so the model generalizes beyond the limited testbed-captured set.

### Cloud-native / MLOps

- **Kubernetes Pod / Sidecar pattern** — A pod is the smallest deployable unit and can contain multiple containers sharing network and volumes. The "sidecar" pattern co-locates a helper container (here, the AI predictor) with the main one (SPGW-U) so it sees the same network namespace without modifying the main container.
- **ConfigMap** — Decouples configuration (IPs, interface names) from the image. The SPGW-U config (`pgwu_sgi_gw`, `net_ue_ip`, etc.) is injected via ConfigMap.
- **PersistentVolumeClaim / NFS provisioner** — PVCs request storage; the NFS provisioner satisfies them with shared NFS volumes, which is how the model `.hdf5` and scaler `.save` files are mounted at `/mnt/` across pods on different nodes.
- **Multus CNI** — Lets a pod attach to multiple networks simultaneously (e.g., `spgwu-net1..net4`), needed because 5G core functions have multiple logical interfaces (S1-U, SGi, Sx, etc.).
- **Kubeflow Pipelines** — A workflow engine for ML on Kubernetes. Components are containers with typed inputs/outputs; `kfp.compiler.Compiler().compile()` turns a Python DSL pipeline into an Argo Workflow YAML that runs as a DAG, with artifacts passed between steps.
- **`kubectl create` vs `apply`** — `create` errors if the resource exists; `apply` is declarative and reconciles state. The deploy script mixes both intentionally — `create` for one-shot resources (namespace, volumes), `apply` for the workload manifests.
- **`kubeadm join` / token-based cluster bootstrap** — How worker nodes register with the control plane; the token + CA hash from `master-init.sh` is what each worker uses to attach.
- **`nodeName` pin** — Forces a pod onto a specific node (used here to keep `oai-du`/`flexran` on the antenna-worker that owns the USRP). Less flexible than `nodeSelector`/affinity but unambiguous.

### Networking / packet capture

- **`pyshark.LiveCapture` + display filter** — Python wrapper over `tshark`. The display filter `(tcp or udp or sip or stun or dtls) and ((ip.src==192.168.3.0/24 and ip.dst==192.168.20.0/24) or ...)` only keeps packets between the app subnet and UE subnet, dramatically reducing parsing overhead.
- **Why sniff at SPGW-U** — In 4G/5G, all UE user-plane traffic transits the SPGW-U/UPF on its way to the data network. Tapping there gives a single observation point for every UE × app combination without touching the radio path.
- **REST as a control plane** — FlexRAN exposes a REST API for slice configuration and stats. The orchestrator uses standard HTTP `GET /stats/` and `POST /slice/enb/-1`, which makes the AI loop language-agnostic and easy to test with `curl`.

### Likely "why did you do X" questions

- **Why CNN-LSTM over plain LSTM?** Multivariate input with 15 correlated features benefits from CNN's local-pattern extraction before the recurrent stage; CNN reduces dimensionality and noise so the LSTM has a cleaner signal.
- **Why a 1-second mini-window and 30-step history?** Traces showed app-level throughput patterns (WebRTC bursts, SIP signaling) become statistically meaningful around 1 s, and 30 s of history captures a full session-level pattern without the model becoming too deep to train.
- **Why threshold-based mapping instead of an end-to-end RL agent?** The mapping is interpretable, debuggable on real radio hardware, and respects FlexRAN's discrete slice-percentage API; an RL agent would need a much longer real-radio training campaign to converge safely.
- **Why a sidecar instead of a separate pod?** Co-locating with SPGW-U gives shared network namespace (sniffer sees user-plane interface directly) and eliminates inter-pod latency for capture.
- **Why Kubernetes for a telecom workload?** Reproducibility, declarative deployment, rolling upgrades, and a clean separation between core network functions and the AI control plane — all without vendor-specific orchestration.
- **What if the prediction is wrong?** The decision policy is bounded (slice percentages are clamped via the discrete level mapping) and re-evaluated every second, so a single bad prediction is corrected in the next loop. The `if slices != curr_slices` guard avoids thrashing FlexRAN with no-op updates.
