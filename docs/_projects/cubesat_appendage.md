# CubeSat Solar Panel Appendage Release System

**Organization:** Grupo de Tecnología Aeroespacial, UTN — HaedoSat Project (2022–2024)
**Recognition:** Best Presentation Award in Satellite Technology — IAA Latin American Small Satellite Conference, 2024
**Tools used:** Solid Edge (CAD), FDM 3D Printing (PLA), Python, Arduino electronics

---

## Context

This project was developed within UTN's HaedoSat program — a 3U CubeSat (10 cm × 10 cm × 30 cm) designed for Earth observation using an optical camera. The satellite requires four deployable solar panels in a petal configuration to meet its 15.27 W power budget. Each panel carries seven high-efficiency solar cells, and the full array delivers approximately 30 W under AM0 conditions.

The catch: those panels must be folded flat against the satellite during launch without activating, then deployed reliably once in orbit. That's the appendage release system problem — and it has to work on the first try.

---

## The mechanism

### Why burn wire over shape memory alloy

The two main candidates for the release mechanism were burn wire triggering and shape memory alloy (SMA/Nitinol) actuators. A weighted decision matrix across cost, reliability, simplicity, and versatility gave burn wire 87.5% versus SMA at 42.5%.

The reasoning: burn wire is extensively documented in CubeSat applications, cheap, and simple to manufacture. SMA requires precise material training, specialized knowledge, and comes at prohibitive cost for a university program. For a first-generation proof of concept, the right answer was the one the team could actually build and validate.

### Hinge and spring design

Each solar panel (10 cm × 30 cm, 150 g) is held by a spring-hinge assembly that deploys it to a final 90° position when the burn wire releases it.

The torsion springs were calculated from first principles: ASTM A228 piano wire, 1.3 mm diameter, spring index 5.2. With a double spring arrangement (one per hinge), the torsional moment is divided between them — 232 Nmm each — improving load distribution and movement stability. The working stress came out at 182 ksi against a yield strength of 1586 MPa for the chosen material. Nine and a quarter coils, spring length 13.2 mm. The springs were then manufactured by a specialized factory to spec.

Two hinge types were built and tested — Type A (single spring) and Type B (double spring) — to compare behavior experimentally.

### Burn wire system

A 0.25 mm nylon thread holds the panel in the stowed position, tied with a surgeon's knot. The release circuit runs a 30 AWG nichrome wire (0.2 mm diameter, 32 mm long) at 7.4 V — the same voltage supplied by the satellite's onboard lithium polymer battery. When the nichrome wire heats to the nylon's melting point (~220°C), the thread burns through and the spring-hinge deploys the panel.

The required heat was calculated via Joule heating and calorimetry: 7.15 J at 1.66 A over 2 seconds through a 1.3 Ω resistance. Because the current exceeded the direct capability of the control electronics, a relay was added to the circuit. A Python/Arduino system triggered the relay and measured the precise time between command and confirmed panel release, detected via a contact sensor.

---

## What I built

My work spanned the full physical development cycle:

**3D design and printing.** The hinge parts and the structural support rails for the test setup were designed in Solid Edge and manufactured by FDM additive manufacturing in PLA. The CAD models for both hinge types were built to match final assembled tolerances — and per the paper's conclusion, they matched what was built precisely.

**Assembly.** I assembled the test structure — an aluminum frame replicating the 3U CubeSat envelope, with four hollow panels fixed to supports, stiffened by the printed PLA rails. The hinges and spring assemblies were integrated onto the structure along with the panel models.

**Integration.** The electronic drive system — regulated power source, relay circuit, Arduino controller — was integrated with the mechanical structure to form a complete test rig. Making the electrical and mechanical sides work together reliably took iteration; the relay solution for current control came directly out of that integration work.

**Testing.** I ran the deployment tests — 25 runs total at the UTN GTA facilities in October 2024, each timed automatically by the Python/Arduino system from command to confirmed panel separation.

---

## Results

Average deployment time: **0.156 seconds**. A control chart with ±2σ limits showed only 1 test outside bounds — 95%+ consistency across all 25 runs.

The first 17 tests clustered within milliseconds of each other, demonstrating high repeatability. A slight increase in variance appeared in later tests, consistent with progressive degradation of the nichrome wire under repeated heating — not a concern for the intended application, since the mechanism fires once and never again in flight.

The actuation and release times came in significantly faster than pre-test estimates. The mechanism is simple, inexpensive, and fully achievable with university-level resources and equipment.

---

## Recognition

The work was presented at the **IAA Latin American Small Satellite Conference (2024)**, where it received the **Best Presentation Award in Satellite Technology** — the top recognition in that category, awarded among researchers and engineers from across Latin America.

---

*The GTA group is currently evaluating NiTiNOL shape memory alloy wires as a next-generation alternative — the burn wire work provides the performance baseline for that comparison.*
