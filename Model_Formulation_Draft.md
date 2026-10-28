---
title: "Mathematical Model Formulation: Scale Formation and Inhibitor Dosing in a Pipeline"
subtitle: "SCE-701 Directed Research, working draft v0.1"
---

# 0. How to read this draft

Every equation is tagged with where it comes from.

- **[Lit]**: taken from a paper and checked on the page image or in the extracted text. Source and equation number are given. Paper IDs (2-09, 1-12, ...) are the file prefixes in `Core_101_Download`; the full list is in `Model_Equation_Matrix.xlsx`.
- **[Std]**: a standard textbook or geometric relation, not taken from our papers. Please sanity-check these.
- **[Prop]**: my own modelling proposal (how the pieces are combined). These are the parts to discuss with your supervisor and the senior PhD student.

Sections 1 to 4 describe the physics model. Section 5 is the estimator (EKF/UKF). Section 6 is the inhibitor controller (MPC). Sections 7 to 9 cover parameters, validation and the gaps.

# 1. Scope and assumptions

- **System:** one pipe segment, length $L$, inner diameter $d_0$, single-phase water, flow rate $Q(t)$, mean velocity $u=4Q/(\pi d^2)$.
- **Scales:** BaSO~4~ is the reference case (most data: 1-07, 1-12, 1-15, 2-09, 5-05). CaCO~3~ and SrSO~4~ reuse the same structure with different constants. Fe scales are a later extension.
- **One limiting ion pair per scale.** For BaSO~4~ the coupled concentration $c$ is the Ba^2+^ or SO~4~^2-^ concentration (1 mol of solid removes 1 mol of each; 2-09 Eq. 14).
- **1D plug flow** with a wall sink (axial dispersion neglected; radial effects enter only through the mass-transfer coefficient).
- **Deposit** is thin, adherent and uniform around the circumference; it changes the flow area and the wall thermal resistance.
- **Inhibitor** is a single sub-stoichiometric scale inhibitor acting by surface adsorption (kink-site Langmuir model, 2-09). 3-16 supports this over a chelation picture.
- **Out of scope for v0.1:** multiphase flow, flashing and CO~2~ partitioning, nucleation in the bulk, particle (adhesion) deposition, porous-medium or near-wellbore domains.

# 2. Variables

| Symbol | Meaning | Unit | Role |
|---|---|---|---|
| $c(x,t)$ | coupled limiting-ion concentration | mol/m$^3$ | state |
| $\delta(x,t)$ | deposit thickness | m | state (hidden) |
| $C_{inh}(x,t)$ | inhibitor concentration | ppm | state |
| $T(x,t)$ | fluid temperature | K | state (optional) |
| $Q(t)$, $T_{in}$, $c_{in}$ | flow, inlet temperature, inlet concentration | | measured inputs |
| $C_{inh,in}(t)$ | injected inhibitor concentration | ppm | **control input** |
| $d(x,t)=d_0-2\delta$ | effective diameter | m | derived |
| $\theta_p$ | parameters (see Section 7) | | estimated |

# 3. Physics model

## 3.1 Thermodynamics: saturation state

Saturation index [Lit: 2-09, definition in text; 5-05 Eq. 1-2]:

$$SI=\log_{10}\frac{IAP}{K_{sp}},\qquad IAP=\gamma_{+}\gamma_{-}\,[M^{2+}][X^{2-}]$$

Solubility product as a function of temperature and pressure [Lit: 5-05 Eq. 3]:

$$\ln K_{sp}=A+\frac{B}{T}+C\ln T+DT+E\,\frac{P-P_{ref}}{P_{ref}},\qquad P_{ref}=0.1\ \text{MPa}$$

Activity coefficients $\gamma_i$ come from an electrolyte model: eNRTL (5-05 Eq. 4-6) or Pitzer (ScaleSoftPitzer in 2-09 and 1-12; PHREEQC in 2-02 and 2-15).

Equilibrium concentration $c_{eq}(T,P,\text{brine})$ is the concentration at $SI=0$, computed by the same solver [Lit: 2-09 p.9; 1-12].

*Design choice [Prop].* Do not solve the activity model inside the filter. Precompute $c_{eq}$ and $SI$ on a grid of $(T,P,\text{composition})$ with PHREEQC or ASPEN, then interpolate. This keeps the estimator fast and uses ASPEN only for what it does well.

## 3.2 Hydraulics and mass transfer

Coupled-ion diffusivity and its temperature dependence [Lit: 2-09 Eq. 2-3]:

$$D_{couple}=\frac{|z_az_b|\,D_aD_b}{|z_a|D_a+|z_b|D_b},\qquad D_{a/b}=\frac{k_BT}{6\pi\mu r_{a/b}}$$

Dimensionless groups [Std]:

$$Re=\frac{\rho u d}{\mu},\qquad Sc=\frac{\mu}{\rho D_{couple}}$$

Sherwood number by flow regime [Lit: 2-09 Eq. 4-9]:

- Laminar, developing, $Re<2300$ (Hausen): $Sh=3.66+\dfrac{0.0668\,Gz}{1+0.04\,Gz^{2/3}}$, with $Gz=\dfrac{d}{l}Re\,Sc$
- Turbulent, $Re>3100$ (Gnielinski): $Sh=\dfrac{(f/8)(Re-1000)Sc}{1+12.7(f/8)^{1/2}(Sc^{2/3}-1)}$, with $f=\big(0.79\ln Re-1.64\big)^{-2}$
- Transitional $2300<Re<3100$ and rough walls: Churchill-type blend (2-09 Eq. 8-9; copy the exact form from page 6 before coding)

Mass-transfer coefficient [Std]:

$$k_m=\frac{Sh\,D_{couple}}{d}$$

Cross-check in the laminar case: $Sh=1.62\,(Re\,Sc\,d/L)^{1/3}$ [Lit: 1-12 Eq. 12].

Pressure drop through the narrowing pipe [Std, Darcy-Weisbach]:

$$\frac{\partial P}{\partial x}=-f_D\,\frac{\rho u^2}{2d}$$

## 3.3 Deposition kinetics (two-phase)

2-09 and 1-15 describe deposition in two phases. **Phase 1:** an induction time while the first layer forms in the stagnant sublayer. **Phase 2:** growth limited by diffusion to the wall at a rate set by the flow regime.

Induction time for BaSO~4~ [Lit: 1-15 Eq. 2; re-check digits in the PDF]. Here $c_{Ca}$ is the calcium molality and $c_{Ba}$, $c_{SO4}$ are the barium and sulfate concentrations:

$$\log t_{ind}=1.52-\frac{10.87}{SI}-\frac{895.7}{T}+\frac{5477}{SI\cdot T}+0.829\,c_{Ca}+0.225\,\log\frac{c_{Ba}}{c_{SO4}}$$

Lag ramp [Lit: 2-02 Eq. 8, applied here to a local lag clock $t_s$, which is my adaptation, [Prop]]. $t_s$ is the time the local $SI$ has stayed above zero:

$$f_d=\begin{cases}0 & t_s\le t_{ind}\\[2pt] 1-\exp\!\big(-(t_s-t_{ind})/\tau\big) & t_s>t_{ind}\end{cases}$$

Surface reaction constant, Arrhenius in temperature [Lit: 2-02 Eq. 7]:

$$k_s=\alpha\exp\!\Big[-\psi\Big(\frac1T-\frac1{293}\Big)\Big]$$

Combined rate constant with mass transfer and surface kinetics in series [Std, series resistance; [Prop] as a choice]:

$$\frac1{k_{eff}}=\frac1{k_m}+\frac1{k_s}$$

In the diffusion-controlled limit $k_{eff}\to k_m$, which is what 2-09 and 1-12 observe in Phase 2.

Inhibitor effect, Langmuir kink-site model [Lit: 2-09 Eq. 11-12]:

$$\vartheta=\frac{K_{eq}C_{inh}}{1+K_{eq}C_{inh}}=\frac{k_{r,0}-k_{r,inh}}{k_{r,0}}\ \Rightarrow\ k_{r,inh}=\frac{k_{r,0}}{1+K_{eq}C_{inh}}$$

The second form is my algebraic rearrangement; see the check in Section 8. Inhibitor also lengthens the induction time: $t_{ind}\to t_{ind}\,g(C_{inh})$. 2-09 refers to semi-empirical models (Dai et al.); the function $g$ is **still missing** (Section 9).

Wall flux [Prop, combining the above]:

$$J(x,t)=f_d\,\frac{k_{eff}}{1+K_{eq}C_{inh}}\,\big(c-c_{eq}\big)_+\qquad (\text{mol m}^{-2}\text{s}^{-1})$$

## 3.4 State equations

Dissolved species, 1D plug flow with wall sink. $4/d$ is the wall-area-to-volume ratio of a circular tube [Std]. The same structure appears in 2-02 Eq. 11-12:

$$\frac{\partial c}{\partial t}+u\frac{\partial c}{\partial x}=-\frac{4}{d}\,J$$

Check against the literature. For steady $J=k(c-c_{eq})$ and constant $d$, $u$, the solution is $c(x)=c_{eq}+(c_0-c_{eq})\exp(-4kx/(ud))$. This is the same as 1-12 Eq. 16 and 2-09 Eq. 13, $\exp(-k\pi d\,x/Q)$, since $Q=u\pi d^2/4$. **The model reduces to the published tube-test equation.**

Deposit growth [Lit: 2-02 Eq. 13 for the form $\partial s/\partial t=R$; molar-volume conversion is Std]:

$$\frac{\partial \delta}{\partial t}=\frac{M_w}{\rho_s}\,J$$

Inhibitor transport with consumption on the surface [Prop; the consumption term $r_{inh}$ is a **gap**]:

$$\frac{\partial C_{inh}}{\partial t}+u\frac{\partial C_{inh}}{\partial x}=-\,r_{inh}(C_{inh},J,d)$$

Energy balance (optional) [Lit: 2-02 Eq. 9-10, 14]. Heat loss to the surroundings through the pipe and scale layers in series; the scale layer adds thermal resistance, which couples $\delta$ back to $T$ and then to $K_{sp}$ and the rate:

$$\rho_wC_w\Big(\frac{\partial T}{\partial t}+u\frac{\partial T}{\partial x}\Big)=k_w\frac{\partial^2T}{\partial x^2}+q_v(T,T_f,\delta)$$

Boundary and initial conditions [2-02 Eq. 15-16 use the same form]:

$$c(0,t)=c_{in}(t),\quad C_{inh}(0,t)=C_{inh,in}(t),\quad T(0,t)=T_{in}(t),\quad \delta(x,0)=\delta_0(x),\ c(x,0)=c_{init}(x)$$

# 4. Discretisation

Grid of $N$ nodes along $x$ (upwind finite volume for the advective terms, explicit or semi-implicit in time). State vector:

$$z=[c_1..c_N,\ \delta_1..\delta_N,\ C_{inh,1}..C_{inh,N}\ (,T_1..T_N)]^\top\in\mathbb R^{3N\ (\text{or }4N)}$$

Since $u$ is of order 0.1 to 1 m/s and deposit growth is hours to days, the system is **stiff in time scales**: $c$ and $C_{inh}$ respond in seconds to minutes, $\delta$ in hours. Use a quasi-steady solution for $c$ at each step (it is algebraic given $\delta$ and $C_{inh}$), and treat $\delta$ and the parameters as the slow states.

# 5. State estimation (EKF/UKF) [Prop, standard filter equations]

Process: $z_{k+1}=F(z_k,u_k;\theta)+w_k$. Measurements $y_k=H(z_k)+v_k$:

| Measurement | Model $H$ | Source |
|---|---|---|
| Differential pressure | $\Delta P=\int_0^L f_D\rho u^2/(2d)\,dx$, $d=d_0-2\delta$ | [Std]; empirical alternative from 3-07 |
| Effluent ion concentration | $c(L,t)$ | 1-12, 2-09 (as in the tube tests) |
| Thickness sensor (ultrasonic, electrical) | $\delta$ at sensor nodes | Block F sensing, not yet in this folder |
| Temperature, flow rate | direct | |
| Residual inhibitor | $C_{inh}$ at sampling points | |

Augmented-state variant: append slowly-varying parameters $\theta=[K_{eq},\alpha,\psi,\tau]$ to $z$ with random-walk dynamics, so the filter tracks them as scale builds (this is the "adaptive" part). Initial covariance $P_0$ and noise matrices $Q_w,R_v$ are tuned on synthetic data first.

# 6. Inhibitor optimisation (MPC) [Prop]

Receding horizon over $N_p$ steps, decision variable $C_{inh,in}(t)$:

$$\min_{C_{inh,in}}\ \sum_{k}\Big[w_1\,C_{inh,in,k}\,Q_k\;+\;w_2\,\max_x\delta_k(x)^2\;+\;w_3\,\Delta C_{inh,in,k}^2\Big]$$

subject to the model of Section 3 and

- minimum inhibitor concentration at the end of the pipe: $C_{inh}(x,t)\ge C_{MIC}$ wherever $SI>0$
- thickness limit: $\max_x\delta\le\delta_{max}$ (or $\Delta P\le\Delta P_{max}$)
- dosing bounds: $0\le C_{inh,in}\le C_{max}$

2-09 states that minimum dosage can be computed for a required protection time or maximum thickness rate, which is the target behaviour; no closed form was in the pages I read.

# 7. Parameters

| Parameter | Value and source | Status |
|---|---|---|
| $K_{sp}$ constants $A$ to $E$ (BaSO~4~) | paper 5-05 Table 5 | not yet read |
| $\gamma$ model (eNRTL or Pitzer) | 5-05, 2-09 | use solver, not hand-coded |
| $D_{couple}$ | $1.11\times10^{-9}$ m$^2$/s at test conditions (2-09 example) | from 2-09 text |
| $\alpha,\ \psi,\ \tau$ | $2.03\times10^{-8}$ m/s, 9950 K, 16 s (2-02, CaSO~4~) | **placeholder, not BaSO~4~** |
| $K_{eq}$ (inhibitor) | fit from 2-09 Table 2 (calcite/DTPMP) and 1-07 (BaSO~4~/PPCA) | to fit (Section 8) |
| $C_{MIC}$ | surface vs bulk differ (1-07) | to extract |
| $M_w$, $\rho_s$ (BaSO~4~) | 233.4 g/mol, about 4.5 g/cm$^3$ | [Std] |
| $r_{inh}$ | none yet | **gap** |

# 8. Validation plan

1. **Reduce to the tube test.** Simulate the model in the constant-coefficient case and reproduce $C_{eff}=C_{eq}+(C_0-C_{eq})\exp(-k_r\pi d\,l/Q)$ (2-09 Eq. 13). Pass criterion: exact agreement.
2. **Mass transfer.** Compare $k_m$ from Section 3.2 with measured $k_r$ in 2-09 Table 2 (calcite, 50 to 90 $^\circ$C) and 1-12 (barite, laminar tubing).
3. **Inhibitor law.** Calcite with DTPMP, 2-09 Table 2, shows $k_{r,0}/k_{r,inh}$:

   | $T$ | $C_{inh}$ (ppm) | $k_{r,0}/k_{r,inh}$ | implied $K_{eq}=(\text{ratio}-1)/C$ (ppm$^{-1}$) |
   |---|---|---|---|
   | 70 $^\circ$C | 0.05 / 0.10 / 0.25 | 2.25 / 3.37 / 6.56 | 25 / 24 / 22 |
   | 50 $^\circ$C | 0.05 / 0.25 | 1.37 / 4.20 | 7.3 / 12.8 |
   | 90 $^\circ$C | 0.05 / 0.10 / 0.25 | 2.33 / 5.07 / 42 | 27 / 41 / 164 |

   This is my own arithmetic from the table values. The simple form $k_r=k_{r,0}/(1+K_{eq}C)$ holds well at 70 $^\circ$C but drifts at 50 and 90 $^\circ$C, so either $K_{eq}$ depends on temperature, the form differs from the paper's actual fit, or the 90 $^\circ$C points are in a different regime. Check this before relying on the dosing law.
4. **Induction time.** Compare Eq. 2 of 1-15 with the experiments in its Table 1.
5. **Estimator test.** Generate synthetic measurements from the model with noise and a hidden $\delta(x,t)$; check the EKF/UKF recovers thickness and $K_{eq}$. Metrics follow Mujahed's validation list: thickness error, location error, false-alarm rate, decision latency, dosing reduction versus a constant-dose baseline.

# 9. Known gaps and decisions needed

1. **Inhibitor consumption** $r_{inh}$ and the induction-time effect $g(C_{inh})$: no equation yet. Look in 2-13 and 3-07, or find the Dai et al. models cited by 2-09.
2. **$C_{MIC}$ definition:** 1-07 reports that the MIC differs between surface and bulk. Decide which one the constraint uses.
3. **Sensor equations** for thickness: need the sensing papers (components 6 and 8, not yet read).
4. **CaCO~3~ and SrSO~4~ constants.** Fe scales: 1-01 gives $K_{sp}(T,I)$ and an Arrhenius rate form to reuse.
5. **Numerics:** stiff time scales (Section 4) need a quasi-steady treatment; to be tested.
6. **Tool choice (ASPEN vs literature).** Suggested split: literature equations for kinetics and deposition (this draft); a solver (PHREEQC or ASPEN eNRTL) only to tabulate $c_{eq}$ and $SI$; HYSYS, if needed, only for hydraulic boundary conditions. Confirm with your supervisor.
7. **Unread papers** that may add equations: 2-03, 2-04, 2-06, 2-07, 2-08, 2-10, 2-11, 2-12, 5-10, 5-12, and the remaining part of 1-12 and 1-07.
8. **Equation-copy checks** still due: 2-09 Eq. 8-10 exact form, 1-15 Eq. 2 digits.

# 10. Mapping to the research objectives (Mujahed deck)

| Objective | Where in this draft |
|---|---|
| Obj 1: multimodal sensor fusion | Section 5 (measurement model $H$) |
| Obj 2: physics-informed spatio-temporal prediction | Sections 3 and 4 (PDE model, state grid) |
| Obj 3: adaptive inhibitor optimisation | Sections 3.3 and 6 (inhibitor law, MPC with MIC) |
