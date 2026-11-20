# CLPS1492-Final-Project

11/20/2029 - Drakeson Hu
Focus is on proper implementation and initial testing of the continuous waveforms for the DG and CA3 paths.
 - the DG path and the CA3 paths now have a working implementation of continuous projection scaling
     - DG and CA3 are both determined by a cosine function, with peaks and troughs aligned to the original on/off phases as the original network
     - the cosine function is wrapped in a tangent function. The n parameter varies how smooth the function is with the parameter k. At k = 0, the function is equivalent to the original function. Increasing k gives a series of functions converging pointwise to the original discrete version, so as k increases, we approximate the discrete, original code.
 - using the tangent wrapper function, we vary the "smoothness" of the wave. Using this parameter, we varied the two pathways' smoothness separately, with 5 values for each pathway. We tested all combinations, giving us a 25-condition sweep.
    - each condition was tested with 100 trials, using a deterministic seed based on trial index. We measured time to convergence and AB recall accuracy on the AB/AC paired task. Results can be found in /results/n100
 - all code was written with Claude Opus 5.5, with details in AI_LOG.md