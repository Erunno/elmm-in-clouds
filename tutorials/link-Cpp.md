# Tutorial: Linking C++ to ELMM Fortran

This tutorial outlines the essential steps to call C++ functions from the ELMM Fortran codebase, including passing arrays. This guide assumes you are working in the `src/` directory and have already created the C++ files.

-----

## Step 1: Create the C++ Source Files

First, create your C++ files. The `extern "C"` block is essential to prevent C++ name mangling, which allows Fortran to find the functions.

### `src/hello_cpp.cpp`

This is a simple "Hello World" function.

```cpp
#include <iostream>

// This extern "C" block is CRITICAL.
extern "C" {
    void hello_from_cpp() {
        std::cout << "===================================" << std::endl;
        std::cout << "    Hello from C++!                " << std::endl;
        std::cout << "===================================" << std::endl;
    }
}
```

### `src/array_test.cpp`

This function accepts a pointer to an array and its dimensions. It's written to expect a `float*` to match Fortran's default single-precision `real(knd)`. It demonstrates correct column-major indexing.

```cpp
#include <iostream>
#include <iomanip> // For formatting output

extern "C" {
    /**
     * Receives a Fortran array.
     * Note: We use 'float*' to match the Fortran 'real(knd)' 
     * which is single precision in the default build.
     */
    void pass_array_to_cpp(float* arr, int rows, int cols) {
        
        std::cout << "--- C++ Function Start ---" << std::endl;
        std::cout << "C++ sees a 1D array of size " << rows * cols << "." << std::endl;
        std::cout << "Iterating to show Fortran's (col-major) memory layout:" << std::endl;

        // Fortran (i, j) maps to C++ index [i + j*rows]
        // We use 0-based indexing here.
        for (int j = 0; j < cols; ++j) {       // Loop over columns
            for (int i = 0; i < rows; ++i) {   // Loop over rows
                
                int index = i + j * rows;

                std::cout << "Fortran index (" << i+1 << "," << j+1 << ") is C++ index [" 
                          << index << "] = " << std::fixed << std::setprecision(1) 
                          << arr[index] << std::endl;
                
                // Modify the array
                arr[index] *= 10.0f;
            }
        }
        std::cout << "C++ multiplied all elements by 10." << std::endl;
        std::cout << "--- C++ Function End ---" << std::endl;
    }
}
```

-----

## Step 2: Update the Build System (src/SConstruct)

You must tell SCons to compile your new C++ files. Add the following lines to `src/SConstruct`, (around line 530) before the `Depends(objs[-1], ...)` line.

```python
# ... (inside src/SConstruct)
 objs += env.Object(target = target_dir + 'momentum_diffusion', source = 'momentum_diffusion.f90',F90FLAGS=env['F90FLAGS'] + [long_lines_flag])

 objs += env.Object(target = target_dir + 'hello_cpp',          source = 'hello_cpp.cpp')
 objs += env.Object(target = target_dir + 'array_test',        source = 'array_test.cpp')
 
 Depends(objs[-1], 'wmfluxes-inc.f90')
# ...
```

-----

## Step 3: Modify the Fortran Code (src/main.f90)

Finally, modify the main Fortran program to define the C++ functions and call them.

### Define the C++ Interfaces

In `src/main.f90`, add the `interface` block. The correct location is **after** `implicit none`.

```fortran
  ! ... (use statements)
  use custom_par

  implicit none

  !=================================================
  ! ADD THIS INTERFACE BLOCK
  !=================================================
  interface
    ! Hello world function
    subroutine hello_from_cpp() bind(C, name="hello_from_cpp")
    end subroutine hello_from_cpp

    ! Array test function
    subroutine pass_array_to_cpp(arr, rows, cols) bind(C, name="pass_array_to_cpp")
      use Parameters, only: knd
      use iso_c_binding, only: c_int

      ! Use dimension(*) for a C-style contiguous pointer
      real(knd), dimension(*), intent(inout) :: arr 

      ! Pass dimensions by value
      integer(c_int), value, intent(in) :: rows, cols
    end subroutine pass_array_to_cpp
  end interface
  !=================================================

  real(knd), allocatable, dimension(:,:,:)   :: U          !Velocity component in x direction -- horizontal
  ! ... (rest of variable declarations)
```

### Call the C++ Functions

In the executable part of `src/main.f90` (e.g., after the `OutTStep` call, around line 80), add the test block.

```fortran
  ! ... (at line ~80)
  call OutTStep(U, V, W, Pr, &
                Temperature, Moisture, Scalar, &
                time_stepping%dt, delta)
  
  !=================================================
  ! C++ Hello World Call
  !=================================================
  call hello_from_cpp()

  !=================================================
  ! C++ Array Test
  !=================================================
  block
    real(knd), dimension(2, 3) :: test_array
    
    ! Initialize a 2-row, 3-column array
    test_array = reshape([1.0_knd, 2.0_knd, 3.0_knd, 4.0_knd, 5.0_knd, 6.0_knd], [2, 3]) 
    
    print *, ''
    print *, '--- Fortran (Before Call) ---'
    print '(3F5.1)', test_array 
    
    ! Pass the address of the first element
    call pass_array_to_cpp(test_array(1,1), 2, 3)
    
    print *, '--- Fortran (After Call) ---'
    print '(3F5.1)', test_array
    print *, ''
  end block
  !=================================================

  init_phase = .false.
  run_phase = .true.
  ! ...
```

-----

## Step 4: Compile and Run

You can now re-compile the model (from within your container) using the same command as before. The build system will pick up the new files and link them correctly.

```bash
# (From within the container, in /opt/build/elmm/src)
unset CC
unset CXX
./make_release
```
