# Cascaded Branch Predictor Implementation

## Overview
The cascaded branch predictor is a predictive model used in computer architecture to improve the accuracy of branch prediction. It combines multiple predictors, typically a bimodal predictor and a two-level predictor, to achieve better prediction rates than using either predictor alone. The cascaded approach leverages the strengths of each individual predictor to handle different types of branch patterns effectively.

This project implements a cascaded branch predictor within the Simplescalar suite.  The implementation of the cascaded predictor is based on the paper "The Cascaded Predictor: Economical and Adaptive Branch Target Prediction" by Karel Driesen and Urs Hölzle from the Department of Computer Science, University of California, which introduces a novel method for adaptive branch target prediction, optimizing prediction accuracy while minimizing resource usage.

## Installation Guide
To run this project, the simple-scalar simulater is required.  You can check the project here: [https://github.com/stevekuznetsov/simple-scalar/blob/master/README]
Once installed, change the files ```bpred.c```, ```bpred.h``` and ```sim-outorder.c``` of the downloaded project for the ones in the folder ```modifications```.

Then you will be able to run simulations using the cascaded predictor indicating the predictor type as ```cascade```.

## Modifications to Simplescalar Suite
The cascaded predictor combines a bimodal predictor and a two-level predictor to improve branch prediction accuracy.

### Bpred.h
Added a new predictor type `BPredCascade`.

### Bpred.c
The main modifications in this file are:
- Added global variables to track predictor state.
- Modified predictor creation to handle the cascade predictor.
- Initialized PHT based on predictor type.

This is done as follows.  In ```bpred_create```, the cascade option is added, creating the bimodal and twolev components:

```
switch (class) { 
case BPredCascade: 
  cascade=1; 
  /* bimodal component */ 
  pred->dirpred.bimod =  
  bpred_dir_create(BPred2bit, bimod_size, 0, 0, 0); 
  /* 2-level component */ 
  pred->dirpred.twolev =  
  bpred_dir_create(BPred2Level, l1size, l2size, shift_width, xor); 
  break;   
  ... 
}
```
When creating the twolev predictor, if the cascade variable is 1, the PHT is initialized to 4, indicating that it's never been used:

```
case BPred2Level: 
    { 
        ... 
     
      if(cascade) 
      { 
         for (cnt = 0; cnt < l2size; cnt++) 
        { 
        pred_dir->config.two.l2table[cnt] = 4; 
        } 
      }  
      else 
      { 
        /* initialize counters to weakly this-or-that */ 
        flipflop = 1; 
        for (cnt = 0; cnt < l2size; cnt++) 
        { 
        pred_dir->config.two.l2table[cnt] = flipflop; 
        flipflop = 3 - flipflop; 
        } 
      }  
      break; 
    }

```

The same applies to the bimodal predictor, but in this case the initialization is 3:
 ```
case BPred2bit:
  ...
    if(cascade) 
    { 
        for (cnt = 0; cnt < l1size; cnt++) 
      { 
        pred_dir->config.bimod.table[cnt] = 3; 
      } 
    } 
    else 
    { 
      /* initialize counters to weakly this-or-that */ 
      flipflop = 1; 
      for (cnt = 0; cnt < l1size; cnt++) 
      { 
        pred_dir->config.bimod.table[cnt] = flipflop; 
        flipflop = 3 - flipflop; 
      } 
    } 
    break;

```
In ```bpred_config```, we add the new entry of the cascaded predictor:
```
 
switch (pred->class) { 
   
    case BPredCascade: 
    bpred_dir_config (pred->dirpred.bimod, "bimod", stream); 
    bpred_dir_config (pred->dirpred.twolev, "2lev", stream); 
    fprintf(stream, "btb: %d sets x %d associativity",  
        pred->btb.sets, pred->btb.assoc); 
    fprintf(stream, "ret_stack: %d entries", pred->retstack.size); 
    break; 
     
    ... 
   }

```
If the predictor is ```comb``` or ```cascade```, we show how many times have the bimodal and the twolev been used:
```
if (pred->class == BPredComb || pred->class == BPredCascade ) 
    { 
      sprintf(buf, "%s.used_bimod", name); 
      stat_reg_counter(sdb, buf,  
               "total number of bimodal predictions used",  
               &pred->used_bimod, 0, NULL); 
      sprintf(buf, "%s.used_2lev", name); 
      stat_reg_counter(sdb, buf,  
               "total number of 2-level predictions used",  
               &pred->used_2lev, 0, NULL); 
    }
```
In ```bpred_lookup```, if the value of the PHT is 4, we use the bimodal and, if not, we use the gshare.  We also need to 
update the variable ```cascadeUtilitzarBimod``` to know which predictor is being used:
```
case BPredCascade: 
  if ((MD_OP_FLAGS(op) & (F_CTRL|F_UNCOND)) != (F_CTRL|F_UNCOND)) 
    { 
      char *bimod, *twolev; 
      bimod = bpred_dir_lookup (pred->dirpred.bimod, baddr); 
      twolev = bpred_dir_lookup (pred->dirpred.twolev, baddr); 
     
      dir_update_ptr->dir.bimod = (*bimod >= 2); 
      dir_update_ptr->dir.twolev  = (*twolev >= 2); 
       
      if (*twolev == 4) 
        { 
          dir_update_ptr->pdir1 = bimod; 
          dir_update_ptr->pdir2 = twolev; 
          cascadeUtilitzarBimod=1; 
        } 
      else 
        { 
          dir_update_ptr->pdir1 = twolev; 
          dir_update_ptr->pdir2 = bimod; 
          cascadeUtilitzarBimod=0; 
        } 
    } 
```
Accordingly, the hit and miss variables are modified to keep track of the predictors accuracy.
Finally, when the bimodal predictor has a miss, the PHT of the gshare is initialized to a value of 2 (taken) or 1 (not taken).
When ghsare is used, we stop updating the bimodal, as it will not be used anymore:
```
if (dir_update_ptr->pdir2) 
{ 
  if(cascade) 
  { 
    if(cascadeUtilitzarBimod && cascadeBimodHaFallat) 
    { 
      if (taken)
      {
        *dir_update_ptr->pdir2=2; 
        else 
        *dir_update_ptr->pdir2=1; 
      } 
    } 
    else 
    {  
      if (taken) 
      {
          if (*dir_update_ptr->pdir2 < 3) 
              ++*dir_update_ptr->pdir2; 
      } 
      else 
      { /* not taken */ 
        if (*dir_update_ptr->pdir2 > 0) --*dir_update_ptr->pdir2; 
      }
    }
  } 
}
```

### Sim-outorder.c
The main modifications in this file are:
- Defined and registered parameters for the cascade predictor.
- Modified options handling to support cascade predictor configuration.

Definition of the parameters for the new predictor:
```
/* cascade predictor config */ 
static int cascade_nelt = 0; 
static int cascade_config[0];
```
Modification of the ```opt_reg_int_list()```:
```
opt_reg_int_list(odb, "-bpred:cascade", 
"cascade predictor config )", 
cascade_config, cascade_nelt, &cascade_nelt, 
/* default */cascade_config, 
/* print */TRUE, /* format */NULL, /* !accrue */FALSE);
````

Add the new option to ```sim_check_options()```:
```
else if (!mystricmp(pred_type, "cascade")) 
{ 
  /* combining predictor, bpred_create() checks args */ 
  if (twolev_nelt != 4) 
  fatal("bad 2-level pred config (<l1size> <l2size> <hist_size> 
  <xor>)"); 
  if (bimod_nelt != 1) 
  fatal("bad bimod predictor config (<table_size>)"); 
  if (cascade_nelt != 0) 
  fatal("bad cascade predictor config (<meta_table_size>)"); 
  if (btb_nelt != 2) 
  fatal("bad btb config (<num_sets> <associativity>)");

  pred = bpred_create(BPredCascade, 
        /* bimod table size */bimod_config[0], 
        /* l1 size */twolev_config[0], 
        /* l2 size */twolev_config[1], 
        /* meta table size */0, 
        /* history reg size */twolev_config[2], 
        /* history xor address */twolev_config[3], 
        /* btb sets */btb_config[0], 
        /* btb assoc */btb_config[1], 
        /* ret-addr stack size */ras_size); 
}
```

## Contributors
- Mireia Gasco Agorreta
- Tarek Ben Hamdouch
