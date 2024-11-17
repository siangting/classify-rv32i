# Assignment 2: Classify
## Part A:
1. Implement abs, argmax, dot, relu, matmul. -> Help with chatGPT 4o


### multiply.s
1. Since multiplication using `mul` is not allowed.
2. Explanation of the algorithm.

```assembly
.globl multiply
.text
# =======================================================
# FUNCTION: Multiply two integers without using 'mul'
# Args:
#   a0: Multiplicand
#   a1: Multiplier
# Returns:
#   a0: Product
# =======================================================
multiply:
    addi sp, sp, -16                 # Save callee-saved registers
    sw   ra, 12(sp)
    sw   s0, 0(sp)
    sw   s1, 4(sp)
    sw   s2, 8(sp)

    mv   s0, zero                    # s0: Product accumulator
    mv   s1, a1                      # s1: Copy of multiplier

                                     # Handle signs
    slt  t0, a0, zero                # t0 = (a0 < 0)
    slt  t1, a1, zero                # t1 = (a1 < 0)
    xor  t2, t0, t1                  # t2 = Sign of result (0: positive, 1: negative)

                                     # Take absolute values
    blt  a0, zero, neg_a0
    j    abs_a0_done
neg_a0:
    sub  a0, zero, a0
abs_a0_done:
    blt  a1, zero, neg_a1
    j    abs_a1_done
neg_a1:
    sub  a1, zero, a1
abs_a1_done:

multiply_loop:
    beq  a1, zero, multiply_end

    andi t3, a1, 1
    beq  t3, zero, skip_add
    add  s0, s0, a0
skip_add:
    slli a0, a0, 1
    srli a1, a1, 1
    j    multiply_loop

multiply_end:
                                     # Apply sign
    bne  t2, zero, neg_result
    j    result_ready
neg_result:
    sub  s0, zero, s0
result_ready:
    mv   a0, s0                      # Return product in a0

    lw   s0, 0(sp)
    lw   s1, 4(sp)
    lw   s2, 8(sp)
    lw   ra, 12(sp)
    addi sp, sp, 16
    jr   ra
```

## Part B:

### read_matrix.s
1. Issue with `fopen`: The program kept exiting with error code 27 when running directly.

   ```
   java -jar ./tools/venus.jar test-src/test_read_1.s 
   Exited with error code 27
   ```
   
   After a long search, the solution was to run:
   
   ```
   bash test.sh test_read_matrix
   ```

2. **Matrix Size Calculation**: Used `jal multiply` to call the external multiply function to calculate the number of elements in the matrix (i.e., rows * columns). This ensures compatibility with architectures that do not support the `mul` instruction directly.

3. **Matrix Pointer Return**: The base address of the matrix is stored in the stack register (36(sp)). Upon returning, it is moved to `a0` to be returned to the caller.
   * **Allocated 56 bytes**:
     * **24 bytes**: Used to store `ra`, `s0` to `s4` (6 registers in total), each occupying 4 bytes.
     * **32 bytes**: Used to store other temporary variables and local variables, such as the number of rows and columns of the matrix, the read buffer, and the allocated address.

4. **`fread` Check**: Properly check whether `fread` has read the expected number of bytes using `bne a0, t0` for verification.

### write_matrix
1. Use the multiply function written above.
2. **Stack Space Adjustment**:
   * The original version had a stack space allocation of `addi sp, sp, -44`, but in the corrected version, it was increased to `addi sp, sp, -48`. This change ensures there is sufficient space for all stored registers and local variables, preventing potential memory overwrites or issues during execution.

### classify.s
1. Fixed `classify.s` to use `multiply` and restore the mode value in `a2` register.
   * Initially made many changes, but found that only fixing the `mul` issue was necessary.
   * From the beginning, the mode value of the `a2` register must be saved, and when determining if the mode is equal to 0, restore this value.

   ```assembly
   sw a2, 48(sp)
   ...
   other code
   ...
   lw a2, 48(sp)
   # If enabled, print argmax(o) and newline
   bne a2, x0, epilogue
   ```
