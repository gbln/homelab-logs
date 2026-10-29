#  Homelab & AI Infrastructure Logs

Welcome to my technical documentation repository. Here I log the setup, optimization, and benchmarks of my homelab, focusing heavily on local Large Language Model (LLM) inference and Proxmox virtualization.

---

## Repository Structure

Each post is organized into its own directory with its respective images and resources in its `assets/` folder:

| Post | Description |
| :--- | :--- |
| **[Post 1: Lenovo P510](./Post1-Upgrading%20a%20Lenovo%20P510%20to%20run%20local%20LLMs/)** | Complete workstation upgrade (Xeon E5-2696 V4, 64GB ECC, Noctua cooling, GPU, 10GB Networking) for homelab. 
| **[Post 2: Radeon Vega VII & llama.cpp](./Post2-Radeon%20Vega%20VII%20&%20llama.cpp/)** | Configuration, compilation with ROCm (`gfx906`), and performance benchmarks for LLMs. |
| **[Post 3: SearXNG & MCP Server](./Post3-SearXNG%20Config%20&%20Setup/)** | Deploying SearXNG in a Proxmox LXC and connecting it via an MCP server (`supergateway`) to give local LLMs internet access. |
| **[Post 4: Claude-Distilled Models](./Post4-Claude-Distilled%20Models%20on%20a%20Radeon%20Vega%20VII/)** | Performance testing and style analysis of Claude-distilled models on the Vega VII. |

---

## Main Hardware Specs

* **Server / Host:** Lenovo ThinkStation P510
* **Processor (CPU):** Intel Xeon E5-2696 V4 (22 cores / 44 threads)
* **RAM:** 64 GB DDR4 ECC
* **AI GPU:** AMD Radeon Vega VII (16 GB HBM2 - 1.02 TB/s bandwidth)
* **System GPU:** NVIDIA P620 2GB
* **Networking:** Intel X520-DA2 10GbE SFP+ network card
* **Storage:** Kioxia Exceria Plus G3 2TB NVMe (via M.2 to PCIe adapter)
* **Hypervisor:** Proxmox VE (LXC & VMs)

---

## Tech Stack
* **LLM Inference:** `llama.cpp` compiled from source with ROCm support.
* **Orchestration / Containers:** Proxmox VE (unprivileged LXCs).
* **Private Web Search:** SearXNG + Nginx + Tailscale.
* **Context Protocols:** Model Context Protocol (MCP) via `supergateway`.

---

## License
This project is licensed under the [MIT](LICENSE) license. You are free to use, modify, and share the documentation with proper attribution.
