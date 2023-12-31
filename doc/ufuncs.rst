Buffered general ufunc explanation
==================================

.. note::

  This was implemented already, but the notes are kept here for historical
  and explanatory purposes.

We need to optimize the section of ufunc code that handles mixed-type
and misbehaved arrays.  In particular, we need to fix it so that items
are not copied into the buffer if they don't have to be.

Right now, all data is copied into the buffers (even scalars are copied
multiple times into the buffers even if they are not going to be cast).

Some benchmarks show that this results in a significant slow-down
(factor of 4) over similar numarray code.

The approach is therefore, to loop over the largest-dimension (just like
the NO_BUFFER) portion of the code.  All arrays will either have N or
1 in this last dimension (or their would be a mismatch error). The
buffer size is B.

If N <= B (and only if needed), we copy the entire last-dimension into
the buffer as fast as possible using the single-stride information.

Also we only copy into output arrays if needed as well (other-wise the
output arrays are used directly in the ufunc code).

Call the function using the appropriate strides information from all the input
arrays.  Only set the strides to the element-size for arrays that will be copied.

If N > B, then we have to do the above operation in a loop (with an extra loop
at the end with a different buffer size).

Both of these cases are handled with the following code::

   Compute N = quotient * B + remainder.
   quotient = N / B  # integer math
   (store quotient + 1) as the number of innerloops
   remainder = N % B # integer remainder

On the inner-dimension we will have (quotient + 1) loops where
the size of the inner function is B for all but the last when the niter size is
remainder.

So, the code looks very similar to NOBUFFER_LOOP except the inner loop is
replaced with::

  for(k=0; i<quotient+1; k++) {
      if (k==quotient+1) make itersize remainder size
      copy only needed items to buffer.
      swap input buffers if needed
      cast input buffers if needed
      call function()
      cast outputs in buffers if needed
      swap outputs in buffers if needed
      copy only needed items back to output arrays.
      update all data-pointers by strides*niter
  }


Reference counting for OBJECT arrays:

If there are object arrays involved then loop->obj gets set to 1.  Then there are two cases:

1) The loop function is an object loop:

   Inputs:
	    - castbuf starts as NULL and then gets filled with new references.
	    - function gets called and doesn't alter the reference count in castbuf
	    - on the next iteration (next value of k), the casting function will
	      DECREF what is present in castbuf already and place a new object.

	    - At the end of the inner loop (for loop over k), the final new-references
	      in castbuf must be DECREF'd.  If its a scalar then a single DECREF suffices
	      Otherwise, "bufsize" DECREF's are needed (unless there was only one
	      loop, then "remainder" DECREF's are needed).

   Outputs:
            - castbuf contains a new reference as the result of the function call.  This
	      gets converted to the type of interest and.  This new reference in castbuf
	      will be DECREF'd by later calls to the function.  Thus, only after the
	      inner most loop do we need to DECREF the remaining references in castbuf.

2) The loop function is of a different type:

   Inputs:

	    - The PyObject input is copied over to buffer which receives a "borrowed"
	      reference.  This reference is then used but not altered by the cast
	      call.   Nothing needs to be done.

   Outputs:

            - The buffer[i] memory receives the PyObject input after the cast.  This is
	      a new reference which will be "stolen" as it is copied over into memory.
	      The only problem is that what is presently in memory must be DECREF'd first.
