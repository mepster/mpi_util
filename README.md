

# mpi_util   -----  Parallelizing RL batch algorithms
                                            by Gary McIntire

    There are multiple ways to do this, but the easiest one(shown here) is to have all 
    processes do their rollouts and then gather those rollouts back to process 0. 
    Let process 0 compute the weight updates, then broadcast those weight updates 
    from rank0 to all the other processes.

    For another example where batches are 'not' accumulated to rank 0, see the trpo_mpi code 
    of the openai baselines https://github.com/openai/baselines/tree/master/baselines/trpo_mpi  
    where the gradients are averaged together.
    That method can require more knowledge of the actual algo whereas accumlating batches 
    to rank0 works for almost all batch RL algorithms
    
    To parallelize your algo, add the mpi_util.py to your project and make a few edits to the 
    original source code following these steps. 
    
    The example parallelized here is the excellent Pat Coady algorithm at ...
  
 &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; <a href="https://github.com/pat-coady/trpo"> https://github.com/pat-coady/trpo</a><br>
    
 Install mpi4py<br>
    &emsp; conda install mpi4py mpich2

1. Tensorflow wants to take all the GPU memory. Fix that by adding a GPU config to the session creators <br>
    Add gpu_options to tf.Session like this ...<br>
    <code>self.sess = tf.Session(graph=self.g, config=mpi_util.tf_config)  # see tf_config in the mpi_util code 
    </code>

2. Add mpi_util<br>
	<code>import mpi_util</code>

3. When nprocs is known shortly after program start, fork nprocs...<br>
	<code>if "parent" == mpi_util.mpi_fork(args.nprocs, gpu_pct=args.gpu_pct): sys.exit()</code>

4. Each process and environment will need different random seeds computed<br>
    <code>mpi_util.set_global_seeds(seed+mpi_util.rank)</code><br>
    <code>env.seed(seed+mpi_util.rank)</code>

5. print statements, env monitoring/rendering, etc may need to be conditional such that only <br>
    one process does it<br>
    <code>if mpi_util.rank == 0: print(...</code><br>
    it can be useful to prepend print statements with the rank   <code>print( str(mpi_util.rank) + ... )</code>

6. Accumulate the batches with something like<br>
    <code>d = mpi_util.rank0_accum_batches({'advantages': advantages, 'actions': actions, 'observes': observes,</code> <br>
    <code>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;'disc_sum_rew': disc_sum_rew}) </code><br>
    <code>observes, actions, disc_sum_rew, advantages = d['observes'], d['actions'], d['disc_sum_rew'], </code><br>
    <code>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;d['advantages']</code>

7. Since this accumulates the batches to rank0, you can avoid processing weight updates on <br>
    the other processes<br>
    <code>if mpi_util.rank==0:</code><br>
        <code>&emsp;&emsp;policy.update(observes, actions, advantages, logger)  # update policy</code><br>
        <code>&emsp;&emsp;val_func.fit(observes, disc_sum_rew, logger)  # update value function</code>

8. After updating the weights on rank0, broadcast the weights from rank0 back to all the other processes<br>
    <code>mpi_util.rank0_bcast_wts(val_func.sess, val_func.g, 'val')</code><br>
    <code>mpi_util.rank0_bcast_wts(policy.sess, policy.g, 'policy')</code><br>
    
9. You should be able to use multiple gpu cards if you have them (untested)<br>
    <code>with tf.device('/gpu:'+str(mpi_util.rank % numGPUcards)):</code><br>
            <code>&emsp;&emsp;main()</code><br>
 
10. You should be able to use multiple computers as well. See mpirun documentation on host file.<br><br><br><br>

    This code is for tensorflow, but a few alterations would allow it to work on theano, pytorch, etc

    The actual speedup from parallelizing is most dependent on the batch size. Larger batch sizes 
    will get closer to linear speedup, but even small batch sizes can triple or quadruple 
    your wall clock speed with nprocs = 8<br><br>
    
    Run it by giving an openai gym environment name<br><br>
        <code>&emsp;&emsp;python train.py Walker2d-v1 --nprocs 8 --gpu_pct 0.08</code><br>
        You may have to reduce the size of the networks with Humanoid if you have a smaller GPU memory<br>
        See --help


<PRE>
<code>
                        Timing  -  on single Xeon 5680 Hex core/twelve thread  GTX 1070
                        
    python ../train.py Walker2d-v1 --nprocs 10 --gpu_pct 0.05  -n 2000

nprocs  steps_per_sec   reward_mean
1            546             641        # reward is highly variable because robot is highly variable
2           1040             544
3           1712             611
4           2014            1220
5           1950            1513
6           2100             860
7           2300             681
8           2400             452
9           2484             359
10          2339             367
11          2384             410
12          2169             336
</code>
</PRE>
