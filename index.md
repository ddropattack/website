<em>
DDRop is a small, low-cost <span class="my-bold">hardware interposer device that can make writes to a server's memory disappear,</span> causing the computer to read old data as if it were newly written.
This trick has serious consequences for confidential computing technologies designed to protect sensitive workloads on shared cloud infrastructure.
DDRop can interfere with protected virtual machines on Intel and AMD platforms and, on Intel TDX, even forge the security evidence used to prove that a virtual machine is trusted.
</em>

#### The missing guarantee: freshness

<div class="row align-items-center mb-4">
  <div class="col-md-2 text-center img-col">
    <i class="fas fa-history section-ico" aria-hidden="true"></i>
  </div>
  <div class="col-md-10">
    To handle large cloud workloads, the memory encryption in <a href="https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html">Intel <abbr title="Trust Domain Extensions">TDX</abbr></a>, <a href="https://www.intel.com/content/www/us/en/products/docs/accelerator-engines/software-guard-extensions.html">Intel Scalable <abbr title="Software Guard Extensions">SGX</abbr></a>, and <a href="https://www.amd.com/en/developer/sev.html">AMD <abbr title="Secure Encrypted Virtualization — Secure Nested Paging">SEV-SNP</abbr></a> keeps data confidential, but does not provide <span class="my-bold">freshness</span>.
    Without it, the processor can confirm that memory is <em>encrypted</em>; it cannot confirm that memory is <em>current</em>. Stale data still decrypts perfectly.
  </div>
</div>

<div class="callout">
  <div class="callout-title"><i class="fas fa-lightbulb me-1"></i> Why does a dropped write matter?</div>
  Think of encrypted memory as a locked notebook with no page numbers or dates.
  If an attacker stops a new line from being written, the old line remains. When you read it later, it still decrypts to valid text.
  The system has no way to tell that the latest update never arrived.
  DDRop uses this gap by silently discarding writes. A protected VM can then continue using old, attacker-chosen contents, while the encryption engine sees nothing wrong.
</div>

DDRop interferes with writes rather than trying to read encrypted data, so memory encryption does not stop the attack. It also uses interfaces that the hypervisor relies on to move protected memory, so the victim VM does not need to contain a software bug. On Intel TDX, we use the attack to put a confidential VM into debug mode and forge attestation reports.

#### A silent switch on the DDR5 bus

<div style="text-align: center; margin-bottom: 1em;">
<img src="assets/images/interposer-front.jpg" style="width: 60%; border-radius: 8px;">
</div>

Earlier DDR5 interposers were passive and required bulky equipment. They also had to slow the memory bus to work with second-hand lab hardware, making the change easier to notice. DDRop is a compact board of analog switches that runs at <span class="my-bold">native DDR5 speed</span> and can be installed in minutes.

DDR5's redesigned command bus makes the address-aliasing tricks used by earlier attacks impractical. DDRop instead interferes with the bus's error handling. The interposer deliberately introduces a parity error and then suppresses the alert. The memory module discards the command, and the processor does not learn that the command was dropped. The write is *gone*.

All schematics, board files, and firmware are released as open source on our [<i class="fab fa-github"></i> GitHub repository](https://github.com/ddropattack/ddrop).

<a class="link-label" data-bs-toggle="collapse" href="#bom" role="button" aria-expanded="false" aria-controls="bom"><i class="fas fa-info-circle"></i> <span class="lbl">What does it cost to build?</span> <i class="fas fa-chevron-down" aria-hidden="true"></i></a>

<div class="table-responsive collapse" id="bom" style="font-size: .8em;">
<table class="table table-hover mx-auto w-100" style="margin: 0px auto; display: table;">
  <thead>
    <tr><th>Component</th><th>Supplier</th><th>Cost</th></tr>
  </thead>
  <tbody>
    <tr><td>Interposer PCB with stencil</td><td>JLCPCB</td><td>$45</td></tr>
    <tr><td>Interposer electronic parts</td><td>Digikey, LOTES</td><td>$30</td></tr>
    <tr><td>Controller board PCB</td><td>JLCPCB</td><td>$4</td></tr>
    <tr><td>Controller electronic parts</td><td>Digikey</td><td>$40</td></tr>
    <tr><td>Teensy 4.1 microcontroller</td><td>Digikey</td><td>$40</td></tr>
    <tr><td><span class="my-bold">Total</span></td><td></td><td><span class="my-bold">$159</span></td></tr>
  </tbody>
</table>
<p style="font-size: .9em; color:#666;">Bill of materials for a single interposer system (at a build quantity of 10). Excludes R&amp;D and assembly labor.</p>
</div>


#### DDRop in action: Breaking Intel TDX
{: #ddrop-in-action}

Intel TDX isolates confidential virtual machines, called Trust Domains (TDs), from a potentially compromised host hypervisor. To prevent the privileged hypervisor from tampering with guest memory translations, TDX delegates address mappings to Secure Extended Page Tables (SEPT), which are encrypted in RAM under the respective TD's key and managed exclusively by trusted firmware (the TDX module).

When the host allocates a new SEPT page (`TDH.MEM.SEPT.ADD`), the TDX module writes empty entries to securely initialize it. DDRop silently drops these writes, so the page keeps pre-crafted stale ciphertext that decrypts into malicious page-table entries. This primitive is 100% deterministic and lets an attacker-controlled TD remap its own memory onto *any physical address* in RAM. 
Under TDX's default <span class="my-bold">logical integrity (LI)</span> mode, this allows the attacker to corrupt the ciphertext of *other* TDs and their control structures. Even TDX's stronger <span class="my-bold">cryptographic integrity (CI)</span> mode would only prevent tampering across different key domains, still allowing an attacker to forge the attestation measurement of their own TD.

<div class="callout my-3">
  <div class="callout-title"><i class="fas fa-unlock me-1"></i> From dropped writes to arbitrary memory access</div>
  Controlling the secure page tables allows an adversary to read from and write to any physical address, including critical TDX metadata like <strong>debug status</strong> (LI) and <strong>launch measurement</strong> (LI + CI).
</div>

<p class="mt-3 mb-2">
  <a class="btn-demo" data-bs-toggle="collapse" href="#case-study-debug" role="button" aria-expanded="false" aria-controls="case-study-debug">
    <i class="fas fa-bug"></i> <span class="lbl">Case Study: Toggling Debug Mode on a Victim TD</span> <span class="btn-demo-mode" title="Works only under logical integrity mode">LI only</span> <i class="fas fa-chevron-down chevron-icon ms-1" aria-hidden="true"></i>
  </a>
</p>

<div class="collapse" id="case-study-debug">
  <div class="case-study-card">
    <p>
      In production, Intel TDX strictly prevents the hypervisor from inspecting a TD's memory. A debug flag in the TD's attributes ensures the host cannot use the debug read/write API. However, the compromised SEPT mappings allow an attacker to corrupt these attributes, including the debug flag. Because this data is encrypted under a different key, it decrypts into pseudorandom noise, giving each attempt a <strong>50% chance of setting the debug bit</strong>. Once active, the hypervisor dumps the victim's plaintext memory via the debug API and then cleanly restores the original captured ciphertext, leaving the victim running unaware with an unmodified attestation status.
    </p>
    <div style="text-align: center; margin-top: 0.85rem;">
      <video 
        controls 
        playsinline 
        preload="metadata" 
        style="width: 100%; border-radius: 8px; display: block; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">
        <source src="assets/images/tdcs-poc.mp4" type="video/mp4">
        Your browser does not support the video tag.
      </video>
    </div>
  </div>
</div>

<p class="mt-3 mb-2">
  <a class="btn-demo" data-bs-toggle="collapse" href="#case-study-mrtd" role="button" aria-expanded="false" aria-controls="case-study-mrtd">
    <i class="fas fa-certificate"></i> <span class="lbl">Case Study: Forging Attestation of an Attacker TD</span> <span class="btn-demo-mode" title="Works under both logical and cryptographic integrity mode">LI + CI</span> <i class="fas fa-chevron-down chevron-icon ms-1" aria-hidden="true"></i>
  </a>
</p>

<div class="collapse" id="case-study-mrtd">
  <div class="case-study-card">
    <p>
      To support remote attestation, the TDX module records the initial state of a TD in a launch measurement, which is signed on request so that a remote tenant can verify what was booted. Here the attacker targets a TD of their own: a <strong>rogue VM they launch and control, running whatever backdoor they like.</strong> Because this TD's control structure (TDCS) is encrypted under the attacker's own key, the injected SEPT entries give direct plaintext write access to it, allowing them to overwrite the launch measurement with that of a legitimate workload. When the rogue VM then requests an attestation report via <code>TDG.MR.REPORT</code>, the TDX module returns a valid report containing the forged measurement, so the backdoored VM passes remote attestation as if it were the trusted one.
    </p>
    <div style="text-align: center; margin-top: 0.85rem;">
      <img src="assets/images/change-mrtd.png" alt="Forged MRTD attestation measurement" style="width: 100%; border-radius: 4px; box-shadow: 0 4px 12px rgba(0,0,0,0.1);">
    </div>
  </div>
</div>

### Questions &amp; Answers
{: #questions-and-answers}

<div class="accordion" id="accordionExample">

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button" type="button" data-bs-toggle="collapse" data-bs-target="#collapseWho" aria-expanded="true" aria-controls="collapseWho">
        Who is behind this research?
      </button>
    </h2>
    <div id="collapseWho" class="accordion-collapse collapse show" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>DDRop is joint work by researchers at KU Leuven, ETH Zurich, Durham University, and Google:</p>
        <ul>
          <li><a href="https://www.esat.kuleuven.be/cosic/people/person/?u=u0156850">Jesse De Meulemeester</a> (COSIC, KU Leuven)</li>
          <li>Stefan Gloor (ETH Zurich)</li>
          <li>Patrick Jattke (ETH Zurich)</li>
          <li><a href="https://moghimi.org/">Daniel Moghimi</a> (Google)</li>
          <li><a href="https://www.durham.ac.uk/staff/david-f-oswald/">David Oswald</a> (Durham University)</li>
          <li><a href="https://www.durham.ac.uk/staff/martin-j-thompson/">Martin Thompson</a> (Durham University &amp; ZF Automotive UK Ltd)</li>
          <li><a href="https://comsec.ethz.ch/kaveh-razavi/">Kaveh Razavi</a> (ETH Zurich)</li>
          <li><a href="https://www.esat.kuleuven.be/cosic/people/person/?u=u0018159">Ingrid Verbauwhede</a> (COSIC, KU Leuven)</li>
          <li><a href="https://vanbulck.net/">Jo Van Bulck</a> (DistriNet, KU Leuven)</li>
        </ul>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseAffected" aria-expanded="false" aria-controls="collapseAffected">
        Should I be worried about my data?
      </button>
    </h2>
    <div id="collapseAffected" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
        For everyday laptops and phones, no: DDRop is a research attack aimed at <span class="fw-medium">confidential-computing servers</span> in the cloud that run Intel TDX, Intel Scalable SGX, or AMD SEV-SNP on DDR5.
        </p>
        <p>
        If you rely on those technologies to protect workloads from the cloud provider itself, this matters: DDRop shows that an attacker with brief physical access can undermine them. We disclosed everything to Intel and AMD in advance under coordinated disclosure.</p>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapsePhysical" aria-expanded="false" aria-controls="collapsePhysical">
        It needs physical access, is that really a threat?
      </button>
    </h2>
    <div id="collapsePhysical" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
        Confidential computing is meant to protect sensitive data, <em>even from the cloud provider</em>. DDRop needs a <span class="fw-medium">brief, one-time visit</span> to install the interposer. Besides that, DDRop assumes the standard TEE threat model, with the adversary controlling the OS/hypervisor and BIOS. Once the interposer is installed, the attack is carried out entirely in software. Possible ways to get that access include:
        </p>
        <div class="threat-grid">
          <div class="threat-grid-container">
            <div class="threat-card">
              <div class="threat-icon"><i class="fas fa-id-badge"></i></div>
              <div class="threat-name">Data-center insiders</div>
              <div class="threat-desc">Rogue technicians, sysadmins, or contractors with server rack access.</div>
            </div>
            <div class="threat-card">
              <div class="threat-icon"><i class="fas fa-boxes"></i></div>
              <div class="threat-name">Supply-chain tampering</div>
              <div class="threat-desc">Interposers planted, or modified modules swapped in, during transit or provisioning.</div>
            </div>
            <div class="threat-card">
              <div class="threat-icon"><i class="fas fa-gavel"></i></div>
              <div class="threat-name">Compelled access</div>
              <div class="threat-desc">Physical hardware access compelled by law enforcement or governments.</div>
            </div>
          </div>
        </div>

        <div class="callout my-2">
          <div class="callout-title"><i class="fas fa-lightbulb me-1"></i> DDRop needs physical access once</div>
          The interposer can be installed in minutes and runs at native DDR5 speeds. After installation, the attack is controlled in software.
        </div>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseWhatCC" aria-expanded="false" aria-controls="collapseWhatCC">
        What is "confidential computing," and who uses it?
      </button>
    </h2>
    <div id="collapseWhatCC" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
         It is a family of hardware features that let you run a workload in the cloud without having to trust the cloud provider.
         Popular platforms include Intel TDX and Scalable SGX, and AMD SEV-SNP.
         The hardware locks memory behind access control and encryption, so a physical adversary sees only scrambled data.
        </p>

        <div class="cc-compare-wrapper">
          <div class="cc-grid">
            <div class="arch-box unsecure">
              <div class="arch-header">
                <span><i class="fas fa-server me-1"></i> Standard Cloud VM</span>
                <span class="badge-status unsecure"><i class="fas fa-unlock"></i> Unrestricted Access</span>
              </div>
              <div class="arch-vm-wrap">
                <div class="vm-card standard">
                  <div class="vm-badge unsecure"><i class="fas fa-unlock me-1"></i> No Hardware Isolation</div>
                  <div class="vm-title"><i class="fas fa-desktop me-1"></i> Standard Guest VM</div>
                  <div class="vm-desc">Data stored unencrypted in host DRAM</div>
                </div>
              </div>
              <div class="arch-flow unsecure">
                <i class="fas fa-arrow-down me-1"></i> Direct memory read / write / inspect access <i class="fas fa-arrow-up ms-1"></i>
              </div>
              <div class="arch-host untrusted-host">
                <div class="host-title"><i class="fas fa-server me-1"></i> Untrusted Host / Hypervisor</div>
                <div class="host-desc">Full control over guest memory, page tables, and execution</div>
              </div>
            </div>

            <div class="arch-box secure">
              <div class="arch-header">
                <span><i class="fas fa-shield-alt me-1"></i> Confidential VM</span>
                <span class="badge-status secure"><i class="fas fa-lock"></i> Protected Data</span>
              </div>
              <div class="arch-vm-wrap">
                <div class="vm-card secure">
                  <div class="vm-badge secure"><i class="fas fa-microchip me-1"></i> Hardware Root of Trust / CPU Boundary</div>
                  <div class="vm-title"><i class="fas fa-user-lock me-1"></i> Confidential VM</div>
                  <div class="vm-desc">Memory transparently encrypted by CPU engine</div>
                </div>
              </div>
              <div class="arch-flow blocked">
                <i class="fas fa-shield-alt me-1"></i> Direct access blocked by hardware memory encryption
              </div>
              <div class="arch-host untrusted-host">
                <div class="host-title"><i class="fas fa-server me-1"></i> Untrusted Host / Hypervisor</div>
                <div class="host-desc">Manages physical server, but cannot read or alter CVM memory</div>
              </div>
            </div>
          </div>
        </div>

        <div class="adopters-bar">
          <div class="adopters-label"><i class="fas fa-check-circle me-1"></i> Where confidential computing is used:</div>
          <div class="adopters-chips">
            <span class="chip"><a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sev-snp.html"><i class="fab fa-aws"></i> AWS</a></span>
            <span class="chip"><a href="https://cloud.google.com/blog/products/identity-security/rsa-snp-vm-more-confidential"><i class="fab fa-google"></i> Google Cloud</a></span>
            <span class="chip"><a href="https://learn.microsoft.com/en-us/azure/confidential-computing/confidential-vm-overview"><i class="fab fa-microsoft"></i> Microsoft Azure</a></span>
            <span class="chip"><a href="https://research.ibm.com/blog/amd-sev-ibm-hybrid-cloud"><i class="fas fa-server"></i> IBM Cloud</a></span>
            <span class="chip"><a href="https://signal.org/blog/private-contact-discovery/"><i class="fas fa-comment-dots"></i> Signal Private Contact Discovery</a></span>
            <span class="chip"><a href="https://engineering.fb.com/2025/04/29/security/whatsapp-private-processing-ai-tools/"><i class="fab fa-whatsapp"></i> WhatsApp Private AI</a></span>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseFix" aria-expanded="false" aria-controls="collapseFix">
        Can a software or firmware update fix it?
      </button>
    </h2>
    <div id="collapseFix" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
        Only partly. Vendors can make the attack harder by restricting the memory-management interfaces DDRop uses, checking that critical writes actually arrived, or looking for the interposer during boot. The paper discusses several defenses.
        </p>
        <p>
        These measures do not add freshness to the memory-encryption hardware. The underlying problem is in <span class="my-bold">hardware</span>: today's scalable memory encryption does not keep track of whether a value is current. A lasting fix would require memory-encryption engines with integrity and freshness.
        </p>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseEnc" aria-expanded="false" aria-controls="collapseEnc">
        Doesn't memory encryption already prevent this?
      </button>
    </h2>
    <div id="collapseEnc" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
          No. Memory encryption on TDX, SEV-SNP, and Scalable SGX protects <strong>confidentiality</strong> against someone reading the memory bus, but it lacks <strong>freshness and replay protection</strong> for this attack: it cannot distinguish a current value from an old one.
        </p>

        <div class="callout my-2">
          <div class="callout-title"><i class="fas fa-lightbulb me-1"></i> Encryption does not tell you whether data is current</div>
          Memory encryption protects the contents, but by itself it does not record whether a value is newer than the one it replaces. Without freshness counters or an integrity structure that detects stale data, an old ciphertext can still be accepted.
        </div>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseTrilogy" aria-expanded="false" aria-controls="collapseTrilogy">
        How is DDRop different from earlier interposer attacks (BadRAM, Battering RAM, WireTap, TEE.fail)?
      </button>
    </h2>
    <div id="collapseTrilogy" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>Physical attacks on encrypted cloud memory fall into two broad groups: attacks that <em>listen</em> to the memory bus and attacks that <em>modify</em> what happens on it. DDRop is the first active hardware interposer for current DDR5 servers.</p>
<div class="lineage">
  <div class="lineage-title">
    <i class="fas fa-headphones lead-ico" aria-hidden="true"></i>
    Passive: listening to the bus
    <span class="sub">observe traffic, then infer secrets via side channels</span>
  </div>
  <div class="chain">
    <div class="stage">
      <div class="smark"><a href="https://www.usenix.org/conference/usenixsecurity20/presentation/lee-dayeol"><img src="assets/images/logo-membuster.svg" alt="Membuster logo"></a></div>
      <div class="yr">2020</div>
      <div class="nm"><a href="https://www.usenix.org/conference/usenixsecurity20/presentation/lee-dayeol">Membuster</a></div>
      <div class="ds">Snoops the unencrypted address of Intel Client SGX to learn access patterns with a high-end logic analyzer.
      </div>
      <div class="props">
        <span class="chip"><i class="fas fa-headphones"></i>Passive</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR4</span>
        <span class="chip"><i class="fas fa-bullseye"></i>Client SGX</span>
        <span class="chip"><i class="fas fa-satellite-dish"></i>Logic analyzer</span>
        <span class="chip"><i class="fas fa-tag"></i>~$170k</span>
      </div>
    </div>
    <div class="stage">
      <div class="smark"><a href="https://wiretap.fail/"><img src="assets/images/logo-wiretap.png" alt="WireTap logo"></a></div>
      <div class="yr">2025</div>
      <div class="nm"><a href="https://wiretap.fail/">WireTap</a></div>
      <div class="ds">Uses a secondhand analyzer for ciphertext side-channel analysis on Scalable SGX platforms with downclocked DDR4 memory.</div>
      <div class="props">
        <span class="chip"><i class="fas fa-headphones"></i>Passive</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR4</span>
        <span class="chip"><i class="fas fa-bullseye"></i>Scalable SGX</span>
        <span class="chip"><i class="fas fa-tachometer-alt"></i>Downclock</span>
        <span class="chip"><i class="fas fa-recycle"></i>Secondhand</span>
        <span class="chip"><i class="fas fa-tag"></i>&lt;$1k</span>
      </div>
    </div>
    <div class="stage">
      <div class="smark"><a href="https://tee.fail/"><img src="assets/images/logo-teefail.png" alt="TEE.fail logo"></a></div>
      <div class="yr">2025</div>
      <div class="nm"><a href="https://tee.fail/">TEE.fail</a></div>
      <div class="ds">Uses a secondhand analyzer for ciphertext side-channel analysis on TDX and SEV-SNP platforms with downclocked DDR5 memory.
      </div>
      <div class="props">
        <span class="chip"><i class="fas fa-headphones"></i>Passive</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR5</span>
        <span class="chip"><i class="fas fa-bullseye"></i>TDX · SEV</span>
        <span class="chip"><i class="fas fa-tachometer-alt"></i>Downclock</span>
        <span class="chip"><i class="fas fa-recycle"></i>Secondhand</span>
        <span class="chip"><i class="fas fa-tag"></i>&lt;$1k</span>
      </div>
    </div>
  </div>
</div>

<div class="lineage">
  <div class="lineage-title">
    <i class="fas fa-bolt lead-ico" aria-hidden="true"></i>
    Active: tampering with the bus
    <span class="sub">alter what the memory sees</span>
  </div>
  <div class="chain">
    <div class="stage">
      <div class="smark"><a href="https://badram.eu"><img src="assets/images/logo-badram.png" alt="BadRAM logo"></a></div>
      <div class="yr">2024</div>
      <div class="nm"><a href="https://badram.eu">BadRAM</a></div>
      <div class="ds">Makes a module lie about its size, which creates a hidden address alias. Patched with boot-time checks.</div>
      <div class="props">
        <span class="chip"><i class="fas fa-bolt"></i>Active</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR4/5</span>
        <span class="chip"><i class="fas fa-bullseye"></i>SEV</span>
        <span class="chip"><i class="fas fa-microchip"></i>SPD chip</span>
        <span class="chip"><i class="fas fa-tag"></i>&lt;$10</span>
      </div>
    </div>
    <div class="stage">
      <div class="smark"><a href="https://batteringram.eu"><img src="assets/images/logo-batteringram.png" alt="Battering RAM logo"></a></div>
      <div class="yr">2025</div>
      <div class="nm"><a href="https://batteringram.eu">Battering RAM</a></div>
      <div class="ds">Switches address aliasing on <em>at runtime</em>, which slips past boot-time checks. Works only for DDR4.</div>
      <div class="props">
        <span class="chip"><i class="fas fa-bolt"></i>Active</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR4</span>
        <span class="chip"><i class="fas fa-bullseye"></i>SGX · SEV</span>
        <span class="chip"><i class="fas fa-toggle-on"></i>Runtime alias</span>
        <span class="chip"><i class="fas fa-tag"></i>&lt;$50</span>
      </div>
    </div>
    <div class="stage now">
      <div class="smark"><img src="assets/images/logo.svg" alt="DDRop logo"></div>
      <div class="yr">2026</div>
      <div class="nm">DDRop</div>
      <div class="ds">Selectively drops writes to break scalable memory encryption on DDR5 lacking freshness.</div>
      <div class="props">
        <span class="chip"><i class="fas fa-bolt"></i>Active</span>
        <span class="chip"><i class="fas fa-memory"></i>DDR5</span>
        <span class="chip"><i class="fas fa-bullseye"></i>TDX · SGX · SEV</span>
        <span class="chip"><i class="fas fa-trash-alt"></i>Write drop</span>
        <span class="chip"><i class="fas fa-tag"></i>&lt;$200</span>
      </div>
    </div>
  </div>
</div>
        <p>The <span class="fw-medium">passive</span> attacks (Membuster, WireTap, TEE.fail) observe traffic and use side channels to infer information. They use slower bus speeds to work with older lab equipment. DDRop is <span class="fw-medium">active</span> and runs at native speed; the paper shows how it can copy plaintext between pages, alter page-table entries, and forge attestation. The defenses discussed in the paper can make the attack harder, but they do not add freshness to the memory-encryption hardware.</p>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseOther" aria-expanded="false" aria-controls="collapseOther">
        Which platforms are affected?
      </button>
    </h2>
    <div id="collapseOther" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <div class="lineage">
          <div class="lineage-title">
            Vulnerable platforms
            <span class="sub">bus-accessible memory without hardware freshness counters</span>
          </div>
          <div class="chain">
            <div class="stage" style="border-left: 4px solid var(--danger);">
              <div class="nm"><i class="fas fa-server me-1"></i> Intel TDX</div>
              <div class="ds">Evaluated on 5th Gen Xeon Scalable server. Affects both CVMs and the TDX Module, so we can force debug mode and forge attestation reports.</div>
              <div class="props">
                <span class="chip"><i class="fas fa-memory"></i>DDR5</span>
                <span class="chip"><i class="fas fa-lock"></i>AES-XTS</span>
                <span class="chip"><i class="fas fa-shield-alt"></i>Optional Integrity</span>
                <span class="chip" style="color: var(--danger-ink); border-color: rgba(164, 87, 76, 0.25); background: rgba(164, 87, 76, 0.07);"><i class="fas fa-times-circle"></i>No Freshness</span>
              </div>
            </div>
            <div class="stage" style="border-left: 4px solid var(--danger);">
              <div class="nm"><i class="fas fa-server me-1"></i> Intel Scalable SGX</div>
              <div class="ds">Evaluated on 5th Gen Xeon scalable processor. The lack of freshness enables dropped writes to force stale data to victim enclaves.</div>
              <div class="props">
                <span class="chip"><i class="fas fa-memory"></i>DDR5</span>
                <span class="chip"><i class="fas fa-lock"></i>AES-XTS</span>
                <span class="chip"><i class="fas fa-shield-alt"></i>Optional Integrity</span>
                <span class="chip" style="color: var(--danger-ink); border-color: rgba(164, 87, 76, 0.25); background: rgba(164, 87, 76, 0.07);"><i class="fas fa-times-circle"></i>No Freshness</span>
              </div>
            </div>
            <div class="stage" style="border-left: 4px solid var(--danger);">
              <div class="nm"><i class="fas fa-server me-1"></i> AMD SEV-SNP</div>
              <div class="ds">Evaluated on EPYC Turin. SEV encrypts memory but has no freshness to detect dropped writes. The relocation API lets us copy arbitrary victim pages.</div>
              <div class="props">
                <span class="chip"><i class="fas fa-memory"></i>DDR5</span>
                <span class="chip"><i class="fas fa-lock"></i>AES-XEX</span>
                <span class="chip" style="color: var(--danger-ink); border-color: rgba(164, 87, 76, 0.25); background: rgba(164, 87, 76, 0.07);"><i class="fas fa-times-circle"></i>No Freshness</span>
              </div>
            </div>
          </div>
        </div>
        <div class="lineage">
          <div class="lineage-title">
            Immune architectures
            <span class="sub">protected by hardware integrity trees</span>
          </div>
          <div class="chain" style="grid-template-columns: 1fr;">
            <div class="stage" style="border-left: 4px solid var(--success);">
              <div class="nm"><i class="fas fa-laptop me-1"></i> Intel Client SGX</div>
              <div class="ds">Older desktop/laptop Intel Core CPUs use a hardware Merkle integrity tree with freshness counters, so the memory controller catches stale data.</div>
              <div class="props">
                <span class="chip"><i class="fas fa-memory"></i>DDR4</span>
                <span class="chip" style="color: var(--success-ink); border-color: rgba(95, 125, 85, 0.25); background: rgba(95, 125, 85, 0.07);"><i class="fas fa-sitemap"></i>Merkle Tree</span>
                <span class="chip" style="color: var(--success-ink); border-color: rgba(95, 125, 85, 0.25); background: rgba(95, 125, 85, 0.07);"><i class="fas fa-history"></i>Freshness</span>
                <span class="chip" style="color: var(--slate); border-color: rgba(122, 106, 85, 0.25); background: rgba(122, 106, 85, 0.07);"><i class="fas fa-ban"></i>Deprecated</span>
              </div>
            </div>
          </div>
        </div>

      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseVendors" aria-expanded="false" aria-controls="collapseVendors">
        What do Intel and AMD say?
      </button>
    </h2>
    <div id="collapseVendors" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>
          We followed coordinated disclosure and shared our hardware, techniques, and proof-of-concept exploits with Intel and AMD ahead of time; we also informed Arm.
          These vendors have acknowledged our findings, but noted that <span class="fw-medium">physical attacks on DRAM are out of scope</span> for their current products.
        </p>
        <p>
         Intel has <a href="https://youtu.be/FqHekMFxmkk">recently characterized</a> research on hardware interposers as "out of scope but not out of mind" and indicated that it is considering next-generation memory-encryption schemes with stronger hardware protections.
         Research such as DDRop matters for understanding the fundamental limitations of today's technology and the attacker capabilities such attacks require, thereby informing the design of more robust memory-encryption schemes.
        </p>
        <p>
          Following a coordinated disclosure on September 14, 2026, both vendors have issued a public security advisory: <a href="https://intel.com/content/www/us/en/security-center/announcement/intel-security-announcement-2026-08-11-001.html">Intel advisory</a> | <a href="https://www.amd.com/en/resources/product-security/bulletin/amd-sb-3048.html">AMD advisory</a>.
        </p>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseTakeaways" aria-expanded="false" aria-controls="collapseTakeaways">
        What is the big-picture takeaway?
      </button>
    </h2>
    <div id="collapseTakeaways" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <div class="takeaways-grid">
          <div class="takeaways-container">
            <div class="takeaway-card">
              <div class="takeaway-num">01</div>
              <div class="takeaway-title">DDR5 does not close the gap</div>
              <div class="takeaway-body">
                DDR5's higher speeds and redesigned command protocol make bus tampering harder, but the paper shows that it is still practical at full speed.
              </div>
            </div>
            <div class="takeaway-card">
              <div class="takeaway-num">02</div>
              <div class="takeaway-title">Encryption needs freshness</div>
              <div class="takeaway-body">
                Encryption alone does not prevent replay. If an attacker can drop writes and the hardware cannot detect stale data, integrity and attestation can be undermined.
              </div>
            </div>
            <div class="takeaway-card">
              <div class="takeaway-num">03</div>
              <div class="takeaway-title">Physical attacks are practical</div>
              <div class="takeaway-body">
                Earlier attacks could require about $170,000 of lab equipment. The DDRop hardware costs less than $200 to build, according to the bill of materials above.
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseLogo" aria-expanded="false" aria-controls="collapseLogo">
        Can I reuse the logo?
      </button>
    </h2>
    <div id="collapseLogo" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        <p>Yes — the logo is released under <a href="https://creativecommons.org/publicdomain/zero/1.0/">CC0</a> (all rights waived).</p>
        <p><a href="assets/images/logo.svg"><i class="fas fa-download"></i> Logo (SVG)</a> &nbsp;·&nbsp; <a href="assets/images/logo.png"><i class="fas fa-download"></i> Logo (PNG)</a></p>
      </div>
    </div>
  </div>

  <div class="accordion-item">
    <h2 class="accordion-header">
      <button class="accordion-button collapsed" type="button" data-bs-toggle="collapse" data-bs-target="#collapseMore" aria-expanded="false" aria-controls="collapseMore">
        Where can I read the details?
      </button>
    </h2>
    <div id="collapseMore" class="accordion-collapse collapse" data-bs-parent="#accordionExample">
      <div class="accordion-body">
        The full details are in the <a href="./ddrop.pdf">research paper</a>, with hardware designs and proof-of-concept code on our <a href="https://github.com/ddropattack/ddrop"><i class="fab fa-github"></i> GitHub repository</a>.
      </div>
    </div>
  </div>

</div>