# The Hallmarks of Cancer [WIP]

We could begin this series of notes from first principles, as is often done in college level classes. However, in my experience, doing so is often unnecessary and sub-optimal. One does not need to deeply understand the principles of organic chemistry and biochemistry in order to understand cancer biology. This is not to say that that knowledge is not important. Rather, we believe that that knowledge can be picked up "just in time" as the need arises.

We will thus adopt a top down approach. That is, we'll first present the high level details of an idea, and then, in an iterative fashion, peel back the layers of abstraction until we arrive at the fundamental principles.

This note is a sweeping overview of the entire field of cancer Biology.  It presents core ideas of the field at the highest possible level. As the title suggests, it's drawn from Hanahan & Weinberg's 2011 seminal paper.  Subsequent notes, drawn from "The Biology of Cancer" chapters, will present the ideas in this note in much greater depth.

## Introduction

What is cancer? At a high level, cancer loosely refers to a group of diseases in which rogue cells revolt against the forces of tissue homeostasis. These revolting cells proceed to grow and divide on their own accord. At some stage, these revolting cells may break away from their tissue compartment to invade neighboring compartments. Often, after invading nearby tissue compartments, these rogue cells invade far away tissues via the blood and lymph systems.

How, exactly, do these, so called, rogue cells "revolt"? Further, what conditions enable their "revolt"? We'll answer the first question by presenting the *hallmarks of cancer*. These are biological capabilities that tumors, and the cells that comprise them, acquire during their development. We'll then answer teh second question by outlining the so called *enabling hallmarks*. These are biological conditions that makes it easier/possible for tumors to revolt.

## The Hallmarks of Cancer

In this section, we present the eight (well, six, plus two) hallmarks of cancer. In presenting each hallmark, we shall focus on three core ideas:

1. First, what does the hallmark entail?
2. Second, how do tumor cells acquire that capability?
3. And finally, why does having that capability constitute a revolt against the forces of tissue homeostasis?

### 1. Sustained Proliferative Signalling

- Core feature of all cancer cells: They just keep growing and dividing. [Contrast with neuron cells in neuro-degenerative diseases]
- How do cells know when to grow, when to divide? Tissue Homeostasis. What is a tissue, really?
- Inducing and Sustaining positively acting growth stimulatory signals
- Induction via autocrine stimulation, reciprocal interaction with stromal cells, elevated expression of growth factor receptors, Somatic mutations leading to constitutively active growth factor receptors  
- Sustained by compromising negative feedback loops that are supposed to dampen growth signals. Examples: mutations in Ras, PTEN Phosphatase

### 2. Evading Growth Suppressors (Removing Checks and Balances)

- Circumvent cellular programs that negatively regulate cell proliferation
- Tumor Suppressor Genes. How they are discovered -- characteristic inactivation in human and mice tumors. Confirmation via gain ad loss of function experiments in mice
- TP53 & Rb proteins.
- RB: Gatekeeper of cll-cycle progression. Integrates signals from intracellular and extracellular and in response decides whether or not the cell should proceed through its growth and division cycles. What are these external/internal signals? How does the Rb protein "communicate" its decision? How does it make its decision?
- TP53 receives inputs from intracellular stress and abnormality sensors. What are examples of abnormalities? TP53 can pause cell-cycle progression until the stressors are dealt with. Alternatively, it can induce apoptosis if the stresses are extreme and unrecoverable.
- Transforming Growth Factor-$\beta$
- Redundancies
- Contact inhibition and Anoikis: $NF2$, $LKB1$

### 3. Resisting Cell Death

- Programmed cel death by apoptosis serves as a natural barrier to the development of cancer. Short lifespan of epithelial cells.
- What triggers apoptosis? DNA damage (TP53), insufficient survival factor signalling (Bim)
- Extrinsic apoptotic pathway - Fas Ligand & Fas Receptor
- Intrinsic Apoptotic Pathway
- Downstream effectors: Caspases 8 and 9
- The Bcl-2 Family of proteins
- How is apoptosis attenuated in the cells of successful tumors?
  - Loss of TP53
  - Increased Expression of anti-apoptotic Bcl family regulators
  - Downregulation of pro-apoptotic members of the Bcl family
  - Increased expression of survival signals
  - short circuiting the extrinsic ligand induced apoptotic pathway
- Beyond Apoptosis: Autophagy
- Beyond Apoptosis: Necrosis

### 4. Enabling Replicative Immortality (Resisting Term Limits)

- Most normal cell lineages in the body only pass through a limited number of successive growth and division cycles. Cancer cells need unlimited replicative potential.
- What imposes the term limits on normal cell lineages? Telomeres, Telomere shortening, Telomerase.
- What happens when telomeres of a given cell become too short?Fusion-Breakage-Cycles, Senescence and telomere induced crisis
- Immortalization
- The Nature of telomeres and telomerase
- How do cancer cells overcome the telomere barrier? - Jettisoning TP53 and its surveillance of genomic integrity
- BFB cycles - increased genomic mutability - acquisition of telomerase function.
- Other functions of telomerase. Non-canonical functions of telomerase

### 5. Inducing Angiogenesis

- Tumors are tissues (That's why they are called neoplasms). They therefore need, just like normal tissues do, a supply of nutrients and oxygen as well as a system to evacuate metabolic waste and $CO_2$
- Angiogenesis - the generation of new blood vasculature. Vasculogenesis
- Angiogenic switch: Embryogenesis vs Adult (Wound healing, Female cycling) vs During tumor progression
- The nature of the angiogenic switch : $VEGF-A \uparrow$, Hypoxia, $FGF$
- The nature of the tumor neovasculature
- Different flavors of the angiogenic switch
- Angiogenesis inhibitors: $TSP-1 \downarrow$, angiostatin, endostatin
- Pericytes
- The role of myeloid derived cells in tumor angiogenesis.

### 6. Activating Invasion and Metastasis

- When cancer becomes malignant and life threatening (Special cases where benign tumors are life threatening: gliomas ...)
- What is meant by invasion? How is this invasion achieved? What's different about cancer cells that allows them to invade when normal cells do not? N-cadherin & E-cadherin
- The EMT program. EMT in tissue repair. EMT in embryonic morphogenesis. EMT Transcription factors: Snail, Slug, Twist
- The invasion metastasis cascade (not a true cascade in the strict mathematical sense): Invasion, Intravasation, Extravasation, Micro-metastatic growth, Colonization
- The role of cross talk between cancer cells and stromal cells in enabling invasion and metastasis
- Macrophages: an example (informal model)
  - matrix degrading enzymes (MMP, Proteases)
  - Cancer Cells produce IL-4. IL-4 activates macrophages
  - Tumor Associated Macrophages (TAMs) supply EGF to cancer cells (in metastatic breast cancer). Cancer cells stimulate TAMs using CSF-1 (Macrophage Colony Stimulating Factor, a cytokine)
- Phenotypes of high grade malignancy do not arise in a cell autonomous manner. Their manifestation cannot be understood solely through the analysis of tumor cell genomes.
- Reversibility of the Invasive Growth Program
- Distinct forms of Invasion
  - Mesenchymal
  - Collective
  - Amoeboid
- Metastatic Colonization
  - Dormant Micrometastases
  - Nutrient Starvation, Autophagy, Reversible dormancy
  - Colonization function $f(\texttt{cancer\_cell}, \texttt{tissue\_micro\_env})$
  - At what point during tumor progression does metastatic dissemination occur?
  - When and where do cancer cells develop the ability to colonize foreign tissues
  
## The Enabling Hallmarks

- Characteristics of tumor cells or the tumor micro environment that allows cancer cells and the tumors they produce to acquire the six hallmark capabilities

### Genomic Instability

- Why is genomic instability needed?
- How do tumor cells make their genomes unstable?
  - Defective DNA repair machinery
  - Loss of TP53
  - Loss of telomeric DNA and the attendant karyotypic chaos
- Recurrent genetic alterations may point to a causal role of particular mutations in tumor pathogenesis

### Tumor Promoting Inflammation

- Tumors as chronic wounds. Densely infiltrated by cells of the innate and adaptive immune systems.
- The duelling roles of the immune cells in tumors: Elimination vs Fostering.
- Tumor associated inflammatory response may have the paradoxical effect of enhancing tumor progression.
- How does inflammation aid in the acquisition of hallmark capabilities?

## A Few Emerging Hallmarks

- Distinct attributes of cancer cells that have been proposed to be functionally important

### Reprogramming Energy Metabolism

- Warburg effect, Aerobic Gycolysis, Increased number of glucose transporters
- What are the mechanisms underlying the switch to aerobic glycolysis
- What is the functional rationale of oxidative glycolysis in cancer cells?
- Where else do we observer aerobic glycolysis? Recall that cancer cells often make use of existing programs
- Intra-tumoral symbiosis: Hypoxic cells use oxygen and secrete lactate. This lactate is preferentially imported by cancer cells with better access to oxygen -- where else is this observed?

### Evading Immune Destruction

- The immune surveillance theory.
- Tumors that manage to grow to macroscopic sizes have somehow managed to avoid detection by the various arms of the immune system or have managed to limit the extent of immunological killing, thereby evading eradication.
- Immunogenecity -- the ability of a substance such as an antigen to provoke an immune response
- How do we know that the immune system acts as a significant barrier to non viral tumor formation and progression?
- Immunoediting; Immunodeficient mice vs Immunocompetent mice
- Evidence from clinical epidemiology: effect of CTL and NK infiltration on prognosis
- What strategies do cancer cells use to evade immune destruction?

### Dimensions of Tumor Heterogeneity

- A word about reductionist science and its limitations
- What do we know? How do we know it? Why is knowing it import?
- Tumors are complex tissues. But, what, precisely, is a tissue?
- Reductionist view of cancer as a collection of relatively homogeneous cancer cells whose entire biology can be understood by elucidating the cell autonomous properties of these cells
- Enumerate the set of important cell types known to contribute in important ways to the biology of tumors
  
**Cancer Cells and Cancer Stem Cells**:  Functional Definition of CSCs. CSCs express markers that are expressed by normal stem cells in the tissue of origin. What is the origin of these CSCs?
**Endothelial Cells & Pericytes**:
**Cells of the Immune System**:
**Cancer Associated Fibroblasts (CAFs)**:
**Stromal Stm and Progenitor Cells**:

## Concluding Remarks

- The perils of a decentralized system of government.
- Cancer does not invent, it usually never invents, it does not have time to invent. It always coopts.
- The limits of reductionists science.

## References

1. [The Hallmarks of Cancer: Next Generation](https://www.cell.com/fulltext/S0092-8674(11)00127-9)
2. The Biology Of Cancer, Robert Weinberg, 2014
