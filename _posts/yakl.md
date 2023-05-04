---
title: Portable C++ Code that can Look and Feel like Fortran Code with Yet Another Kernel Launcher (YAKL)
author: Matthew Norman
tags: "performance portability", CUDA, HIP, SYCL, OpenMP
---

With the large array of hardware available today, portable code isn't an easy thing to accomplish. It has different meanings, and there are many avenues to seek portability. Here, let's define portability as the ability to run a **single source code** on as many different hardware via different compilation targets.

There are many ways to approach this, the most common being directives-based specifications (e.g., [OpenACC](https://www.openacc.org/) and [OpenMP](https://www.openmp.org/)); Domain Specific Languages, DSLs, valid for certain discretization and algorithmic choices; and portable C++ libraries. The focus of this blog post is on the latter.

There are several C++ portability libraries available, the two most popular likely being [Kokkos](https://github.com/kokkos/kokkos) and [RAJA](https://github.com/LLNL/RAJA). These each some with large features sets and large development teams and are excellent choices for portability. There are several aspects that can be difficult to deal when when porting a Fortran code to portable C++, however. That's where [Yet Another Kernel Launcher (YAKL)](https://github.com/mrnorman/YAKL) comes in.

First, let's dive briefly into what portable C++ actuall **is**.

## What is Portable C++?

First, portable C++ is just a C++ library, not a new specification or language extention / augmentation. It works fully within existing ISO C++ standards and compiler implementations. There are backends that use vendor-specific languages and libraries. However, user-facing code is purely C++, and when using non-accelerator backends, the library is entirely ISO C++.

**Encapsulating code as an object**: First, the C++ language can encapsulate sections of code as an object that can be passed around. The most convenient way to do this is with a C++ lambda express, or an unnamed class that overloads `operator()` to execute some code. This class object can then be passed to the appropriate backend (e.g., a GPU language like CUDA or HIP) to be launched in a manner appropriate for that backend.

For example, if you wanted to launch some code with OpenMP directives, you could write the following directly:
```C++
#pragma omp parallel for
for (int i=0; i < n; i++) { a(i) = i; }
```

Or, you could encapsulate that code as an object and pass it to the backend of your choice:
```C++
launch_kernel( n , [=] (int i) { a(i) = i; } );
```

And you could specify `launch_kernel` to launch that same code in a lot of different ways:
```C++
#if defined(_USE_OPENMP_)
  template <class CODE> void launch_kernel(int n, CODE const &code) {
    #pragma omp parallel for
    for (int i=0; i < n; i++) ( code(i); }
  }
#if defined(_USE_OPENACC_)
  template <class CODE> void launch_kernel(int n, CODE const &code) {
    #pragma acc parallel loop
    for (int i=0; i < n; i++) ( code(i); }
  }
#if defined(_USE_SERIAL_)
  template <class CODE> void launch_kernel(int n, CODE const &code) {
    for (int i=0; i < n; i++) ( code(i); }
  }
#endif
```

This way, we're still writing the exact same loop body, but we can run it in many different ways. C++ portability libraries have more complexity than this, but this is the gist of what's going on. If you want to run on a GPU, compile with `-D_USE_OPENACC_`. If you want threading, compile with `-D_USE_OPENMP_`.

## YAKL Makes Fortran Code Porting Easier

There are some difficulties you'll encounter when attempting to port Fortran code to C++.
* Fortran contains multi-dimensional arrays as a part of the language standard while C++ does not.
* Fortran uses so-called "column-major" index ordering where the first array index varies the fastest, while in C/C++, so-called "row-major" index ordering is used where the last array index varies the fastest.
* Fortran array indices have lower bounds that default to one but can be arbitrary. C/C++ arrays have lower bounds of zero and cannot change.
* Fortran supports creating so-called "automatic" arrays inside subroutines and functions, and compilers often place this data on the stack of the routine for quick allocation. Therefore, patthers that look like frequent allocation and deallocation is very common in Fortran and need to be supported cheaply.

These are where YAKL comes into play. YAKL has multi-dimensional array objects just like Kokkos and RAJA do. YAKL, however, also has Fortran-style multi-dimensional `Array` objects that have column-major index ordering and arbitrary lower bounds that default to one. They also support contiguous slicing to get subsets of the data.

YAKL also has a non-blocking, automatically enabled, and user-transparent pool allocator that optimally uses pre-allocated memory to allow very cheap allocaitons and deallocations on GPU devices. These allocations are extremely expensive to perform without a pool allocator. This way, developers can maintain the look and feel of frequent allocations and deallocations during runtime.

YAKL also contains a limited library of Fortran intrinsics functions like `size`, `maxval`, `minloc`, and `allocated`, which are all frequently called in Fortran code. YAKL also has Fortran hooks to its init and finilize routines as well as its pool allocator. Therefore, the Fortran code can directly allocate a varaible via YAKL and obtain that variable as a contiguous pointer. YAKL can allocate it with Managed or Unified memory, and one can build YAKL to automatically inform OpenMP target and OpenACC runtimes that the Managed or Unified memory variable is already on the device so that the underlying runtime handles the copying to and from device transparently. This way, developers can seamlessly and incrementally share data and port code one small section at a time to portable C++.

With these features, one can port a Fortran code to portable C++ much more quickly and with far less bug potential.

## Some Advantages over Fortran

* **Complex Data Structures**: There are even some potential advantages to using YAKL instead of Fortran. Directives-based approaches in Fortran often have some measure of difficulty in the presence of derived types with type-bound procedures. It varies from compiler to compiler, but difficulties are common. In C++, classes have pedantically defined behavior for classes in terms of how they exist, how they are copied and moved, and how they are created and destroyed. This makes it much easier for vendor languages like CUDA, HIP, and SYCL to maintain clear and predictable behavior in the presence of classes.
* **Inlining**: C++ natively has an easier time inlining code than Fortran does as a language. The reason for this is the module-based structure employed by many Fortran programs. It makes it difficult for a compiler to traverse the module dependencies and find the appropriate code to inline it. In directives-based languages, calling routines from inside kernels tends to be one of the buggier features of the specification, at least as implemented in many compilers. In C++, any code that is visible in the same translation unit via `#include` directives can be easily inlined, and often is inlined, especially in accelerator language implementations like CUDA, HIP, and SYCL.
* **Less Code**: With basic type templating, which is fairly easy to learn, users can write significantly less code when performing the same operation on potentially different types of data. Fortran does not have this feature at this time, and it leads to complex interface blocks and significant amounts of replicated code.
* **More Reliable Debugging**: In Fortran, if you explicitly state the lower and / or upper bounds at a subroutine, the compiler takes those bounds as absolute truth, no matter what. Therefore, if you make a mistake in that pattern, the compiler's bounds checking will not detect the error. In YAKL, however, lower and upper bounds of a multi-dimensional `Array` objects are **tied to the object itself**. Therefore, they can never be overridden, and incorrect indices will be caught every time by the time by the YAKL implementation when `-DYAKL_DEBUG` is defined as a CPP variable during compilation. 
* **Lower Bounds Aren't Erased**: One slightly annoying feature of the Fortran standard is that if you pass an array with non-"one" lower bounds to a function, those bounds get "reset" to lower bounds of one. In YAKL, however, since all of this is tied to the `Array` object itself, the lower bounds are maintained through function interfaces.

## Example Fortran to YAKL Code Conversion

Suppose we start with the following sample code to compute the maximum stable time step for a Cartesian Shallow-Water model in Fortran with OpenACC directives:
```Fortran
function max_stable_dt(height, u, v, cfl, grav, dx, dy)  result(dt)
  real(8), dimension(:,:), intent(in) :: height, u, v
  real(8)                , intent(in) :: cfl, grav, dx, dy
  real(8)                             :: dt
  integer                             :: i, j, nx, ny
  real(8)                             :: gw, dtloc, eps
  nx = size(height,1)
  ny = size(height,2)
  dt = huge(height)      ! Initialize to a large value
  eps = epsilon(height)  ! To avoid division by zero
  !$acc parallel loop collapse(2) present(height,u,v) reduction(min:dt)
  do j = 1 , ny
    do i = 1 , nx
      gw = sqrt(grav * height(i,j))  ! Speed of gravity waves
      ! Compute local maximum stable time step
      dtloc = min( cfl * dx / ( abs(u(i,j)) + gw + eps ) , &
                   cfl * dy / ( abs(v(i,j)) + gw + eps ) )
      ! Compute global minimum of the local maximum stable time steps
      dt = min( dt , dtloc )
    enddo
  enddo
endfunction max_stable_dt
```

If we were to convert this to portable C++ with Fortran-style YAKL, the code would become the following:
```C++
// The lines before the function would be placed in a header file somewhere else
// using and typedef allow the latter code to be more readable
using yakl::fortran::Bounds;
using yakl::fortran::parallel_for;
using yakl::intrinsics::minval;
typedef double real;
typedef yakl::Array<real const,2,yakl::memDevice,yakl::sytleFortran> realConst2d;
typedef yakl::Array<real      ,2,yakl::memDevice,yakl::sytleFortran> real2d;     

// Here begins the main user-level YAKL code:
real max_stable_dt(realConst2d height, realConst2d u, realConst2d v, 
                   real cfl, real grav, real dx, real dy) {
  int nx = size(height,1);
  int ny = size(height,2);
  real eps = epsilon(height); // To avoid division by zero
  real2d dt2d("dt2d",nx,ny);  // Allocate array to store the local max stable time steps
  // do j = 1 , ny
  //   do i = 1 , nx
  parallel_for( "Max stable timestep" , Bounds<2>(ny,nx) , YAKL_LAMBDA (int j, int i) {
    real gw = sqrt(grav * height(i,j));  // Speed of gravity waves
    // Compute local maximum stable time step
    dt2d(i,j) = min( cfl * dx / ( abs(u(i,j)) + gw + eps ) ,
                     cfl * dy / ( abs(v(i,j)) + gw + eps ) );
  });
  // With the local max stable time steps stored, compute the minimum among all of them
  return minval( dt2d );
}
```

Here, we see use of intrinsics like `size` and `maxval`. The code inside the loops is practically identical with the only exception that the reduction needs to be done afterward on temporarily stored values in an `Array` object. 

## More information

For more information on YAKL, please see the [documentation](https://github.com/mrnorman/YAKL/wiki) and the [publication](https://link.springer.com/article/10.1007/s10766-022-00739-0). 

For more information on other C++ libraries, please see the [Kokkos](https://github.com/kokkos/kokkos), [RAJA](https://github.com/LLNL/RAJA), and [SYCL](https://www.khronos.org/sycl/) libraries and associated documentation.


