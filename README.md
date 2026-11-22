# CLPS1492-Final-Project

11/20/2026 - Drakeson Hu
Focus is on proper implementation and initial testing of the continuous waveforms for the DG and CA3 paths.
 - the CA1 inputs and the DG-CA3 paths now have a working implementation of continuous projection scaling
     - both paths are both determined by a cosine function, with peaks and troughs aligned to the original on/off phases as the original network
     - the cosine function is wrapped in a hyperbolic tangent (tanh) function. The n parameter varies how smooth the function is with the parameter k. At k = 0, the function is equivalent to the pure cosine function. Increasing k gives a sequence of functions converging pointwise to the original discrete version, so as k increases, we approximate the discrete, original code.
 - using the tanh wrapper function, we vary the "smoothness" of the wave. Using this parameter, we varied the two pathways' smoothness separately, with 5 values for each pathway, and one additional control condition for the original discrete code vs our approximately discrete version. Thus, we get 6 conditions per pathway. We tested all combinations, giving us a 36-condition sweep.
    - each condition was tested with 100 runs, using a deterministic seed based on trial index. We measured time to convergence and AB recall accuracy on the AB/AC paired task. Results can be found in ch7/hip/results/n100
 - all code was written with Claude Opus 5.5, with details in AI_LOG.md

11/21/2026 - Drakeson Hu
Xcal implementation into network. Changed the learning rule from snapshot-based plus-minus phase error calculation to xcal. 
 - We swept Mtau values for xcal and found that optimal Mtau values massively reduce the performance difference between the discrete and continuous theta models.
 - results can be found in ch7/hip/results/xcal