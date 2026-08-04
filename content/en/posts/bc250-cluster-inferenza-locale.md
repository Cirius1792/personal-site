---
title: "Local Inference on an Asrock BC250 Cluster"
subtitle: "45GB of distributed VRAM across 3 nodes with llama.cpp"
date: 2026-08-04
lastmod: 2026-08-04
draft: false
author: "Ciro Lucio Tecce"
authorLink: "https://ciroluciotecce.it"
description: "How I built a cluster of three Asrock BC250s for local LLM inference: sub-€1000 ex-mining hardware, 45GB of distributed VRAM via RPC, and llama.cpp benchmark results."
license: ""
images: []
tags: ["homelab", "llm", "llama.cpp", "local inference", "bc250", "vulkan"]
categories: ["Homelab", "LLM"]
featuredImage: ""
featuredImagePreview: ""
hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false
toc:
  enable: true
  auto: true
---

## Intro
I've always been a tech enthusiast, and a few years ago I fell into the homelabbing rabbit hole. With all the AI hype around, developing an interest in local inference felt like the next natural step. 

But even beyond the technical challenges and fascination this space brings, I strongly believe that the future of AI is local. This is both for practical reasons regarding the sustainability of the AI ecosystem as a whole—which currently consumes resources completely out of proportion to the actual returns companies get based on pricing—and for the democratization of access to these tools, which cannot and should not remain exclusively in the hands of big external providers. 

Over the past few months, I also think there's a growing awareness that we're becoming increasingly dependent, in one way or another, on artificial intelligence for the support and speed it gives us in everyday tasks—currently mostly at work. But I don't think we're too far off from its increasingly pervasive role in our personal lives as well. 

That's why I started looking for a budget-friendly solution that would let me take my first steps into local inference, experimenting and learning a thing or two about deploying these kinds of services on distributed infrastructure. 
The first thing I did was pick my target: at least 50GB of VRAM to host at least two models in parallel: 
1. a stronger model with decent reasoning capabilities for designing and supervising daily coding tasks 
2. an "executor" model, less capable at reasoning but faster at running plans pre-defined by the main model 

The choice fell on Qwen3.6 27B as the "smart" model and Qwen3.6 35B-A3B as the executor.

Unthinkable to go for a single graphics card with all that VRAM: I had to look for ways to split the load across multiple cards/machines.

The priority was still to keep the budget tight, given the current state of the electronics market. This led to my first sacrifice: no vLLM. 

vLLM is leaning more and more towards the enterprise world and is currently, to my knowledge, the most effective and efficient open-source tool for model serving in multi-user parallel scenarios. 

Unfortunately, vLLM doesn't support the Vulkan backend, which happens to be the gateway to this world on a modest budget, since it's supported by virtually any GPU of any tier and age. Furthermore, while vLLM is indeed more efficient when serving multiple concurrent users, at least as of this writing it delivers slightly lower single-user performance than llama.cpp. Consequently, since I was already looking at non-cutting-edge hardware and was, in practice, the sole user of the service, I had to drop the idea of using vLLM—even though it would have better aligned with my goal of picking up a skill to take back to my professional work. 

On the hardware front, I set myself a budget of around €1,000, and I must admit it wasn't easy finding something that met my VRAM requirements in that price range. Skipping all the options I considered along the way for brevity's sake, let's jump straight to the two finalists: 
1. AMD Mi50 32GB: a card from a few years back, lacking support or optimizations for matrix operations (which LLMs benefit immensely from), but with a very impressive memory bandwidth
2. an Asrock BC250 cluster: I mainly owe the idea to [this post](https://www.reddit.com/r/LocalLLaMA/comments/1tj4unp/amd_bc250_and_the_search_for_cheap_compute/) about unlocking the full compute power of this card and its relatively low cost

After comparing both options, I went with the second one, fully aware that performance would take a hit: reaching my target VRAM meant spreading across four separate nodes connected via Ethernet, rather than two cards fitted into the same host as with the Mi50s. This option was also the cheaper of the two, which likely weighed heavily on my decision. Going with two Mi50s would have cost about 50% more once you factored in the supporting hardware I'd need to buy.

Indeed, multi-GPU inference on a single machine would have required a motherboard and CPU with enough PCIe lanes to satisfy the setup's appetite—plus the matching RAM—and given today's prices, that would have cost me a kidney. 

## Hardware and Assembly

First off, let me clarify what we're talking about: the BC250 is not a traditional GPU you plug into a PCIe slot, but a compact motherboard born for crypto mining, built around the AMD Cyan Skillfish APU—a stripped-down version of what's inside the PlayStation 5. In practice, inside every card is a complete computer: 6 Zen 2 cores at 3.5GHz, an RDNA 2 (more or less) GPU with 24 Compute Units (unlockable up to 40 with the mod I discuss in the software section), and 16GB of GDDR6. So every board is effectively its own standalone node, with its own CPU, memory, and network port—not just simple GPUs connected together, which explains many of the design choices (and limits) we'll see.

The key factor for inference is memory: shared between CPU and GPU over a 256-bit bus, with bandwidth around 448 GB/s. LLMs are thirsty for memory bandwidth far more than pure compute power, which is precisely why these ex-mining boards have become popular as a cheap local inference platform. The flip side: memory has to be partitioned in BIOS (I use 512MB dedicated to the GPU and the rest as unified memory via GTT), storage is reduced to a single M.2 slot with just two PCIe 2.0 lanes, networking is Gigabit Ethernet, and software support is Linux-only—on Windows the GPU is unsupported, video encoding/decoding is disabled, and IOMMU is broken by design. To top it off, TDP reaches up to 220W with passive cooling meant for a server rack, not a desktop: which foreshadows the thermal issues I discuss later.

Here is the haul: 
1. 4 BC250 boards 
2. 4 second-hand SSDs picked up on Vinted for €10 each 
3. a 1000W power supply, also second-hand
4. an unmanaged network switch bought on AliExpress for around €20 
5. a custom 3D-printed rack derived by modifying a LabRax
6. two 120mm Arctic fans—where I sacrificed a chunk of my phalanx during test assembly—to cool the whole thing 


{{< image src="/rack_picture.jpg" caption="The 3D-printed rack with the four BC250s" >}}

Note: Unfortunately, one of the four boards never showed signs of life and, mea culpa, I tested them too late after delivery to open a dispute with the seller. Therefore, the results reported below refer to a setup with only three nodes.

The SSDs are neither particularly large nor particularly fast. As for capacity, I plan to replace them all in the future if this setup survives the test of time. Right now, on practically every model swap, the slice deployed to each node is sent sequentially to all cluster nodes over 1Gb Ethernet; with larger SSDs, I could use llama.cpp's RPC caching and avoid resending models over the network. As for model load times from SSD, though, there's not much to be done: the BC250 offers an M.2 2280 PCIe 2.0 slot with only 2 dedicated lanes, so don't expect miracles.
In short, model loading won't be lightning-fast, and that's just something I'll have to live with. 

The four boards are mounted inside a 3D-printed rack based on the LabRax design [LabRax 10" Server Rack - Bolted Version - 5U](https://makerworld.com/en/models/1464819-lab-rax-10-server-rack-bolted-version-5u). I modified and printed two longer crossbars to fit them: otherwise they would never have fit in a traditional 10" rack. 
The boards were mounted vertically side by side in pairs of two, held by two custom brackets I designed myself with my terrible 3D modeling skills after several (failed) iterations and wrong measurements.

{{< image src="/cad.png" caption="The custom bracket for vertical board mounting" >}} 
Final assembly isn't the simplest and has to happen in a specific order: for instance, the boards must be mounted before closing the top of the rack to have enough working room when inserting and tightening the mounting screws. 
[Here](https://www.printables.com/model/1799662-4-bc250-10-rack-mount) you can find everything you need and assembly instructions.

### Thermals
The part that scared me most was thermals: these boards are far from energy-efficient—after all, they were intended for actual servers with proper server-grade cooling. I naively hoped I could cool two of them with a single 120mm Arctic fan placed at the rear, centered between the two cards. 
Poor fool. 
Even so—by upgrading to the PRO version of the same fan, which hits 3000 RPM compared to the previous 1800, and designing and 3D printing a terrible but seemingly somewhat effective fan duct to the best of my abilities—I managed to get thermals down to fairly acceptable levels for my use case. 
So in the current setup, a single fan running at 100% can cool two boards. The right-hand board (facing the front of the rack) likely receives most of the airflow, staying six or seven degrees cooler than the left one. 
This results in the following thermal behavior: 
1. Idle: right board 47°C, left board 55°C
2. Load under distributed inference: right board 66°C, left board 75°C
3. Load under single-node inference: ~85°C

{{< image src="/dashboard_1_node.png" caption="Temperature dashboard — single node" >}}

{{< image src="/dashboard_2_nodes.png" caption="Temperature dashboard — two nodes" >}}

I reported results for both distributed and single-node inference because in the former, since parallelism is implemented at the model layer level in this setup, the boards don't stay continuously active throughout the entire task. Instead, they take turns, looping load from one board to the next. This actually gives the cards time to cool down between turns, keeping overall average temperatures lower. 
That doesn't happen with single-node inference: the load is continuous, and the cards push right up to the dissipation limits of my tiny rack. 

Bottom line: not an amazing result, but the boards aren't reaching core meltdown temperatures. So for now: Mission Accomplished! Fully aware that there's still plenty of room for improvement. 

## Software and Configuration 
Most of the information that helped me fine-tune the cluster setup comes from excellent documentation aggregating the main guides and instructions for this card: [BC250](https://elektricm.github.io/amd-bc250-docs/)

The main configurations applied are: 
1. Ubuntu Server installation (not Fedora, which would have been the better choice, but I'm more familiar with Ubuntu) via a headless custom ISO prepared with Cubit
2. Kernel 7.x installation (built-in with Ubuntu 26.04+)
3. 40 CU unlock across all three boards 
4. VRAM set to 512MB to leave as much system RAM available as possible
5. GTT set to 15GB to allow maximum unified memory usage by llama.cpp
6. MESA drivers installation as outlined in the documentation mentioned above 
7. Docker installation and deployment of llama.cpp built with RPC and Vulkan flags enabled 

All nodes serve strictly as workers: a llama-swap instance on a dedicated host orchestrates model allocation and distribution across the nodes.

Distributed inference is achieved via layer parallelism, which carries less network overhead when connected over standard networking without more sophisticated techniques like dedicated RDMA NICs.
In RPC mode with layer parallelism, llama.cpp doesn't duplicate the model across all nodes; instead, it slices it up: each node handles a subset of the network layers. However, at every generation step, nodes must exchange data across network layer boundaries, and you pay for that traffic with every single produced token. The more nodes you add, the more coordination and synchronization overhead you introduce—which is why, as we'll see, the two-node setup ends up faster than the three-node setup.

To simplify things, all host configuration was applied using Ansible playbooks. 

Specifically, the 40 CU unlock role was completely rewritten (thanks AI!) because the original script doesn't work natively on Ubuntu. 

 <details>
 <summary>Ansible role for the 40 CU unlock</summary>

```yaml
---
# 40 CU Unlock — Inference Cluster Worker Node
# Upstream-aligned build workflow, with Ansible-owned enable/reboot:
#   ./scripts/bc250-enable-40cu.sh build
#   write /etc/modprobe.d/bc250-40cu.conf
#   ansible.builtin.reboot

- name: 40CU | Skip — role is disabled (cu40_enabled = false)
  ansible.builtin.debug:
    msg: "40 CU Unlock is disabled. Set cu40_enabled=true to enable."
  when: not (cu40_enabled | bool)
  tags: [40cu]

- name: 40CU | Converge unlock state
  when: cu40_enabled | bool
  tags: [40cu]
  block:
    - name: 40CU | Compute kernel source paths
      ansible.builtin.set_fact:
        _cu40_kernel_source_version: "{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_dir: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_compat_link: "/usr/src/linux-{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_amdgpu_check: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c"
        _cu40_kernel_source_tree_check: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}/drivers/gpu/drm/amd/amdkfd/Makefile"

    - name: 40CU | Install role dependencies
      ansible.builtin.apt:
        name:
          - build-essential
          - zstd
          - git
          - curl
          - pciutils
          - vulkan-tools
          - mesa-vulkan-drivers
          - "linux-headers-{{ ansible_kernel }}"
          - "linux-source-{{ _cu40_kernel_source_version }}"
        state: present
        update_cache: true
        cache_valid_time: 3600
      become: true

    - name: 40CU | Prepare kernel source tree for upstream build script
      ansible.builtin.shell:
        cmd: |
          set -euo pipefail
          mkdir -p {{ _cu40_kernel_source_dir | quote }}
          for tarball in \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.xz" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.bz2" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.gz"; do
            if [ -f "$tarball" ]; then
              tar xf "$tarball" \
                -C {{ _cu40_kernel_source_dir | quote }} \
                --strip-components=1 \
                --wildcards \
                '*/drivers/gpu/drm/amd/*'
              test -f {{ _cu40_kernel_source_amdgpu_check | quote }}
              test -f {{ _cu40_kernel_source_tree_check | quote }}
              exit 0
            fi
          done
          echo "Cannot find linux-source tarball for {{ _cu40_kernel_source_version }} under /usr/src" >&2
          exit 1
        executable: /bin/bash
        creates: "{{ _cu40_kernel_source_tree_check }}"
      become: true

    - name: 40CU | Check kernel source compatibility link
      ansible.builtin.stat:
        path: "{{ _cu40_kernel_source_compat_link }}"
      register: _cu40_kernel_source_compat_link_stat
      become: true

    - name: 40CU | Create kernel source compatibility link for upstream script
      ansible.builtin.file:
        src: "{{ _cu40_kernel_source_dir }}"
        dest: "{{ _cu40_kernel_source_compat_link }}"
        state: link
        force: true
      become: true
      when: >-
        (not _cu40_kernel_source_compat_link_stat.stat.exists)
        or (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Check existing non-link compatibility source tree
      ansible.builtin.stat:
        path: "{{ _cu40_kernel_source_compat_link }}/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c"
      register: _cu40_kernel_source_compat_tree_stat
      become: true
      when:
        - _cu40_kernel_source_compat_link_stat.stat.exists
        - not (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Assert existing compatibility path is a usable source tree
      ansible.builtin.assert:
        that:
          - _cu40_kernel_source_compat_tree_stat.stat.exists
        fail_msg: >-
          {{ _cu40_kernel_source_compat_link }} exists but is neither a symlink
          managed by this role nor a usable kernel source tree containing
          drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c.
      when:
        - _cu40_kernel_source_compat_link_stat.stat.exists
        - not (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Preflight — detect BC250 PCI device
      ansible.builtin.command:
        cmd: lspci -nn
      register: _cu40_lspci
      changed_when: false
      become: true

    - name: 40CU | Assert BC250 PCI ID 13fe is present
      ansible.builtin.assert:
        that:
          - _cu40_lspci.stdout is search('(?i)13fe')
        fail_msg: >-
          BC250 PCI ID 13fe not found in lspci output. Aborting 40 CU unlock.
        success_msg: "BC250 PCI ID detected (13fe)."

    - name: 40CU | Clone/update upstream repository
      ansible.builtin.git:
        repo: "{{ cu40_repo }}"
        dest: "{{ cu40_clone_dir }}"
        version: "{{ cu40_repo_version }}"
        update: true
        force: true
        depth: 1
      become: true

    - name: 40CU | Verify upstream script exists and is executable
      ansible.builtin.stat:
        path: "{{ cu40_script_path }}"
      register: _cu40_script_stat
      become: true

    - name: 40CU | Assert upstream script is ready
      ansible.builtin.assert:
        that:
          - _cu40_script_stat.stat.exists
          - _cu40_script_stat.stat.executable
        fail_msg: "Upstream script {{ cu40_script_path }} missing or not executable."

    - name: 40CU | Apply upstream script sentinel compatibility fix
      ansible.builtin.replace:
        path: "{{ cu40_script_path }}"
        regexp: "grep -q 'bc250-cc-clear'"
        replace: "grep -q 'bc250-40cu-enable'"
      become: true

    - name: 40CU | Check governor service is active
      ansible.builtin.command:
        cmd: "systemctl is-active {{ cu40_governor_service }}"
      register: _cu40_governor_active
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Read governor config
      ansible.builtin.slurp:
        src: "{{ cu40_governor_config_path }}"
      register: _cu40_governor_config
      become: true

    - name: 40CU | Decode governor config
      ansible.builtin.set_fact:
        _cu40_governor_config_content: "{{ _cu40_governor_config.content | b64decode }}"

    - name: 40CU | Assert governor profile is capped for 40 CU unlock
      ansible.builtin.assert:
        that:
          - _cu40_governor_active.rc == 0
          - _cu40_governor_config_content is search('(?m)^max\s*=\s*1500\s*$')
          - (_cu40_governor_config_content | regex_findall('(?m)^\[\[safe-points\]\]\s*$') | length) == 3
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*350\s*\n\s*voltage\s*=\s*700')
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*1500\s*\n\s*voltage\s*=\s*900')
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*2000\s*\n\s*voltage\s*=\s*1000')
        fail_msg: >-
          Governor safety profile is not in the required 40 CU state.
          Expected max=1500 and safe-points 350/700 + 1500/900 + 2000/1000.

    - name: 40CU | Check if current-kernel amdgpu module is patched
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Ensure build state directory exists
      ansible.builtin.file:
        path: /var/lib/bc250-40cu-unlock
        state: directory
        owner: root
        group: root
        mode: "0755"
      become: true

    - name: 40CU | Check role build marker for current kernel
      ansible.builtin.stat:
        path: "/var/lib/bc250-40cu-unlock/{{ ansible_kernel }}.build-ok"
      register: _cu40_build_marker
      become: true

    - name: 40CU | Check persistent upstream config contains bc250_cc_write_mode=3
      ansible.builtin.shell:
        cmd: >-
          test -f {{ cu40_conf_path | quote }}
          && grep -Eq 'bc250_cc_write_mode=3' {{ cu40_conf_path | quote }}
      register: _cu40_config_enabled
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Check Vulkan/RADV reports num_cu = 40
      ansible.builtin.shell:
        cmd: RADV_DEBUG=info vulkaninfo --summary 2>&1
      register: _cu40_vulkan_summary
      changed_when: false
      failed_when: false
      become: true
      when: cu40_verify_vulkan | bool

    - name: 40CU | Set Vulkan state fact
      ansible.builtin.set_fact:
        _cu40_vulkan_num_cu_40: >-
          {{
            (cu40_verify_vulkan | bool)
            | ternary(
                ((_cu40_vulkan_summary.stdout | default('')) is search('num_cu\s*=\s*40')),
                true
              )
          }}

    - name: 40CU | Compute whether upstream build is required
      ansible.builtin.set_fact:
        _cu40_build_required: >-
          {{
            (_cu40_module_patched.rc != 0)
            or ((not (_cu40_build_marker.stat.exists | default(false)))
                and (not (_cu40_vulkan_num_cu_40 | bool)))
          }}

    - name: 40CU | Reset prepared kernel source tree before rebuild
      ansible.builtin.shell:
        cmd: |
          set -euo pipefail
          for tarball in \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.xz" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.bz2" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.gz"; do
            if [ -f "$tarball" ]; then
              tar xf "$tarball" \
                -C {{ _cu40_kernel_source_dir | quote }} \
                --strip-components=1 \
                --wildcards \
                '*/drivers/gpu/drm/amd/*'
              test -f {{ _cu40_kernel_source_amdgpu_check | quote }}
              test -f {{ _cu40_kernel_source_tree_check | quote }}
              exit 0
            fi
          done
          echo "Cannot find linux-source tarball for {{ _cu40_kernel_source_version }} under /usr/src" >&2
          exit 1
        executable: /bin/bash
      become: true
      when: _cu40_build_required | bool

    - name: 40CU | Build patched module (upstream build)
      ansible.builtin.command:
        cmd: "{{ cu40_script_path }} build"
        chdir: "{{ cu40_clone_dir }}"
      environment:
        MAKEFLAGS: "-j{{ cu40_build_jobs }}"
      register: _cu40_build
      become: true
      when: _cu40_build_required | bool

    - name: 40CU | Mark whether build executed
      ansible.builtin.set_fact:
        _cu40_build_ran: "{{ _cu40_build is defined and _cu40_build is changed }}"

    - name: 40CU | Compute whether enable/reboot is required
      ansible.builtin.set_fact:
        _cu40_enable_required: >-
          {{
            (_cu40_build_ran | bool)
            or (_cu40_config_enabled.rc != 0)
            or (not (_cu40_vulkan_num_cu_40 | bool))
          }}

    - name: 40CU | Verify installed module is patched before enabling
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched_before_enable
      changed_when: false
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Assert installed module can accept 40 CU parameter
      ansible.builtin.assert:
        that:
          - _cu40_module_patched_before_enable.rc == 0
        fail_msg: >-
          Cannot enable 40 CU mode because the installed amdgpu module does not
          expose bc250_cc_write_mode. The upstream build/install step did not
          leave a usable patched module for the current kernel.
      when: _cu40_enable_required | bool

    - name: 40CU | Manage persistent 40 CU modprobe config
      ansible.builtin.copy:
        dest: "{{ cu40_conf_path }}"
        content: |
          # BC-250 40 CU re-enablement
          # Managed by Ansible — 40cu-unlock role
          options amdgpu bc250_cc_write_mode=3
        owner: root
        group: root
        mode: "0644"
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Reboot to load patched amdgpu with 40 CU mode
      ansible.builtin.reboot:
        msg: "Rebooting to load patched amdgpu with bc250_cc_write_mode=3."
        pre_reboot_delay: "{{ cu40_reboot_pre_delay }}"
        reboot_timeout: "{{ cu40_reboot_wait_timeout }}"
        post_reboot_delay: "{{ cu40_reboot_wait_delay }}"
        reboot_command: "{{ (cu40_reboot_command | length > 0) | ternary(cu40_reboot_command, omit) }}"
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Re-check module patched state (post-reconnect)
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-check persistent config (post-reconnect)
      ansible.builtin.shell:
        cmd: >-
          test -f {{ cu40_conf_path | quote }}
          && grep -Eq 'bc250_cc_write_mode=3' {{ cu40_conf_path | quote }}
      register: _cu40_config_enabled_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-check Vulkan/RADV state (post-reconnect)
      ansible.builtin.shell:
        cmd: RADV_DEBUG=info vulkaninfo --summary 2>&1
      register: _cu40_vulkan_summary_post
      changed_when: false
      failed_when: false
      become: true
      when: cu40_verify_vulkan | bool

    - name: 40CU | Re-check governor service state (post-reconnect)
      ansible.builtin.command:
        cmd: "systemctl is-active {{ cu40_governor_service }}"
      register: _cu40_governor_active_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-read governor config (post-reconnect)
      ansible.builtin.slurp:
        src: "{{ cu40_governor_config_path }}"
      register: _cu40_governor_config_post
      become: true

    - name: 40CU | Decode governor config (post-reconnect)
      ansible.builtin.set_fact:
        _cu40_governor_config_content_post: "{{ _cu40_governor_config_post.content | b64decode }}"
        _cu40_vulkan_num_cu_40_post: >-
          {{
            (cu40_verify_vulkan | bool)
            | ternary(
                ((_cu40_vulkan_summary_post.stdout | default('')) is search('num_cu\s*=\s*40')),
                true
              )
          }}

    - name: 40CU | Assert final desired state
      ansible.builtin.assert:
        that:
          - _cu40_module_patched_post.rc == 0
          - _cu40_config_enabled_post.rc == 0
          - (not (cu40_strict_verify | bool)) or (_cu40_vulkan_num_cu_40_post | bool)
          - _cu40_governor_active_post.rc == 0
          - _cu40_governor_config_content_post is search('(?m)^max\s*=\s*1500\s*$')
          - (_cu40_governor_config_content_post | regex_findall('(?m)^\[\[safe-points\]\]\s*$') | length) == 3
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*350\s*\n\s*voltage\s*=\s*700')
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*1500\s*\n\s*voltage\s*=\s*900')
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*2000\s*\n\s*voltage\s*=\s*1000')
        fail_msg: >-
          Final 40 CU verification failed.
          Required: persistent config bc250_cc_write_mode=3, Vulkan num_cu = 40,
          governor service active and capped profile present.

    - name: 40CU | Write role build marker for current kernel
      ansible.builtin.copy:
        dest: "/var/lib/bc250-40cu-unlock/{{ ansible_kernel }}.build-ok"
        content: |
          kernel={{ ansible_kernel }}
          script={{ cu40_script_path }}
          repo={{ cu40_repo }}
          version={{ cu40_repo_version }}
        owner: root
        group: root
        mode: "0644"
      become: true
      when: _cu40_build is defined and _cu40_build is changed

    - name: 40CU | Diagnostic — runtime bc250_cc_write_mode sysfs
      ansible.builtin.shell:
        cmd: cat /sys/module/amdgpu/parameters/bc250_cc_write_mode 2>/dev/null || echo unavailable
      register: _cu40_diag_sysfs_mode
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — last active_cu_number log line
      ansible.builtin.shell:
        cmd: dmesg 2>/dev/null | grep 'active_cu_number' | tail -1 || true
      register: _cu40_diag_active_cu
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — recent bc250-40cu log lines
      ansible.builtin.shell:
        cmd: dmesg 2>/dev/null | grep 'bc250-40cu' | tail -n 8 || true
      register: _cu40_diag_patch_logs
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — upstream status output
      ansible.builtin.command:
        cmd: "{{ cu40_script_path }} status"
        chdir: "{{ cu40_clone_dir }}"
      register: _cu40_diag_status
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Show diagnostics summary
      ansible.builtin.debug:
        msg:
          - "sysfs bc250_cc_write_mode: {{ _cu40_diag_sysfs_mode.stdout | default('n/a') | trim }}"
          - "dmesg active_cu_number: {{ _cu40_diag_active_cu.stdout | default('n/a') | trim }}"
          - "dmesg bc250-40cu lines:\n{{ _cu40_diag_patch_logs.stdout | default('n/a') }}"
          - "upstream status:\n{{ _cu40_diag_status.stdout | default('n/a') }}"

    - name: 40CU | Warn when Vulkan verification is disabled (emergency mode)
      ansible.builtin.debug:
        msg: >-
          WARNING: cu40_verify_vulkan=false. Vulkan/RADV num_cu verification is
          skipped in this run. Use only for emergency diagnostics.
      when: not (cu40_verify_vulkan | bool)
```
 </details>

## Performance

The main question this whole project had to answer was simple: was it worth it? To find out, I ran a series of benchmarks using llama-bench, comparing the two-node configuration against the three-node configuration across four models: Gemma 4 12B, Gemma 4 26B-A4B, Qwen 3.6 35B-A3B, and Qwen 3.6 27B, all in UD-Q4_K_XL quantization (the "XL" variant keeps top-level tensors at higher precision).

The benchmark setup: Vulkan backend with RPC for multi-node distribution, all layers offloaded to GPUs (`-ngl 99`), batch size 4096, and micro-batch 2048. For each model, I measured two phases—prompt processing (512 tokens) and token generation (128 tokens)—each at two context depths: cold (no prefilled tokens, `-d 0`) and full context (4096 prefilled tokens, `-d 4096`). All run via the `bench.sh` script included at the bottom of this section.

The current setup uses *layer parallelism* managed automatically by llama.cpp based on model size and node capacity: each GPU processes a subset of the model's layers, communicating with the other nodes at every step. This detail is important because it explains the main takeaway: **the two-node configuration is faster than the three-node configuration across all 16 comparisons**, though this was an expected outcome. Distributed inference inherently introduces network overhead—every inter-node coordination step comes at a cost—and the third node brings more compute parallelism but also more RPC traffic and synchronization. The fact that degradation is steeper in token generation than in prompt processing stems from the autoregressive nature of LLM inference: the prompt is processed in a single bulk compute run (so network overhead is paid only once), whereas token generation produces one token at a time, demanding continuous step-by-step node coordination.

### Context depth

The first finding concerns context depth, which impacts the two phases in opposite ways. Processing a prompt with the KV cache already filled to 4096 tokens is slower—unsurprising, since attention has to operate on an already populated window—but token generation at depth 4096 gets faster, likely because the extra compute power outweighs the introduced overhead. The charts below show the percentage change relative to depth 0.

{{< image src="/three-node-depth-comparison.png" caption="Context depth effect on three nodes" >}}

{{< image src="/two-node-depth-comparison.png" caption="Context depth effect on two nodes" >}}

| Model | pp512 (3 nodes) | tg128 (3 nodes) | pp512 (2 nodes) | tg128 (2 nodes) |
|---|---|---|---|---|
| Gemma 4 12B | -14.0% | +0.7% | -14.2% | +22.6% |
| Gemma 4 26B-A4B | -22.3% | -1.9% | -24.8% | +13.0% |
| Qwen 3.6 35B-A3B | -19.8% | +16.2% | -18.8% | +28.9% |
| Qwen 3.6 27B | -10.0% | +10.8% | -10.3% | +12.4% |
| **Average** | **-16.5%** | **+6.5%** | **-17.0%** | **+19.2%** |

On average, prompt processing takes roughly a 16-17% hit at full context across both setups, whereas token generation benefits: +19.2% on two nodes, but only +6.5% on three. And on three nodes, the behavior isn't even uniform: Gemma 4 12B stays practically flat, while Gemma 4 26B-A4B actually gets slower.

{{< image src="/long_run.png" caption="Performance scaling with growing context depth" >}}

The outcome is no surprise, but rather confirmation of what you'd expect from layer parallelism over 1 Gb Ethernet: adding a third node brought overhead, not throughput. The charts and table below show this in detail.

{{< image src="/two-vs-three-node-comparison.png" caption="Two vs three nodes comparison" >}}

The following table shows the percentage change moving from two to three nodes (negative values = three nodes are slower).

| Model | Depth | pp512 | tg128 |
|---|---|---|---|
| Gemma 4 12B | 0 | -9.4% | -8.1% |
| Gemma 4 12B | 4096 | -9.1% | -24.5% |
| Gemma 4 26B-A4B | 0 | -9.8% | -32.7% |
| Gemma 4 26B-A4B | 4096 | -6.9% | -41.6% |
| Qwen 3.6 35B-A3B | 0 | -8.7% | -18.2% |
| Qwen 3.6 35B-A3B | 4096 | -9.8% | -26.2% |
| Qwen 3.6 27B | 0 | -3.3% | -4.6% |
| Qwen 3.6 27B | 4096 | -3.0% | -6.0% |
| **Average** | | **-7.5%** | **-20.2%** |

On average, prompt processing drops by 7.5% and generation by 20.2%, peaking at -41.6% for Gemma 4 26B-A4B generation at full context. 

### What this means in practice

With this hardware and setup, running **Qwen 3.6 35B-A3B** distributed across two nodes delivers acceptable performance for day-to-day use on small coding tasks, largely thanks to its prompt processing speed—which, in my view, plays a particularly crucial role in these workflows. Remember that the original goal of this cluster was to deploy Qwen 3.6 35B across two of the four cards and Qwen 3.6 27B across the other two: I'd say the first objective has been met.

Unfortunately, the results for the 27B dense variant are less rosy. Token generation speed is decent enough for everyday use in my opinion, but prompt processing speed leaves much to be desired. Consider that just the initial prompt of a coding agent can easily reach between 8K and 30K tokens, and naturally, as context grows, processing speed degrades significantly, leading to wait times that are unbearable for daily use.

Not to mention that deploying both models with a satisfying context window (between 60K and 80K tokens) simply isn't possible on the three remaining boards in my setup.

### The benchmark script

Here is `bench.sh`, the script I used to collect the benchmark numbers: it executes llama-bench inside the inference engine container, targeting the cluster workers via RPC.

<details>
<summary>bench.sh</summary>

```bash
#!/bin/bash

set -o pipefail

LOGFILE="llama-bench-$(date +%Y%m%d-%H%M%S).log"

docker run --rm \
  --network host \
  --user "$(id -u):$(id -g)" \
  -v "/opt/data/llm/models:/models:ro" \
  --entrypoint /app/llama-bench \
  my-gitea-host/clt/llama-cpp-rpc:latest \
  -rpc 192.168.31.50:50052,192.168.31.52:50052,192.168.31.53:50052 \
  -m /models/gemma/qat/gemma-4-12B-it-qat-UD-Q4_K_XL.gguf \
  -m /models/gemma/qat/gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf \
  -m /models/qwen/3.6/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -m /models/qwen/3.6/Qwen3.6-27B-UD-Q4_K_XL.gguf \
  -d 0,4096 \
  -ngl 99 \
  -b 4096 \
  -ub 2048 \
  --progress \
  2>&1 | tee "$LOGFILE"

exit ${PIPESTATUS[0]}
```

</details>

## Conclusion

The project's goal was simple yet non-trivial to achieve: on a budget around €1,000, bring home at least 50GB of distributed VRAM to serve two models in parallel—a "smart" one with decent reasoning capabilities for coding tasks and a faster executor—while picking up hands-on experience deploying inference services on distributed infrastructure. Choosing an Asrock BC250 cluster proved to be the cheapest option capable of meeting the memory requirement, though certainly not the best in terms of performance.

**What worked.** Unfortunately, due to the prematurely deceased node, I only reached 45GB of VRAM instead of the theoretically possible 60GB. Qwen 3.6 35B-A3B distributed across two nodes delivers usable daily performance, mainly thanks to prompt processing speed, which is crucial for coding tasks. The 40 CU unlock, thermal management (acceptable, albeit riding near the limit), and full setup reproducibility via Ansible—making every node rebuildable straight from the repo—were also successes.

**What worked less well.** The dense Qwen 3.6 27B variant disappoints on prompt processing: with real-world coding agent contexts exceeding 30K tokens, wait times become intolerable. Furthermore, on the three boards left after the dead-on-arrival unit, hosting both models with a satisfying context window just isn't happening. The structural limits highlighted by the benchmarks were also confirmed: networking causes a third node to add overhead rather than throughput, and model loading remains sluggish due to storage bottlenecks.
On top of that, the current software stack—llama-swap in front of llama.cpp—doesn't utilize VRAM optimally: hosting two models in parallel that should theoretically fit, even if snugly, becomes tricky. I frequently ran into OOM issues during heavy parallelism testing.

Was it worth it? I'm not entirely sure: honestly, I don't know if I'll keep this setup around. For under €1,000 I have a three-node cluster that, even if not for distributed inference, could always be repurposed for other distributed computing workloads—after all, we're talking about three Single Board Computers, not just three graphics cards. I'm not quite sure what to do with it long-term; for now, it's a visually pleasing addition to my setup. It was a great opportunity for hands-on experience with the realities and trade-offs of distributed inference that I couldn't have gained otherwise, and I learned a lot along the way: for now, I'm satisfied.
