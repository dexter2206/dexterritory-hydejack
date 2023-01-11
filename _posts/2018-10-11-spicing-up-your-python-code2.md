---
layout: post
title: "Spicing up your Python with a little bit of Fortran - part 2"
description: "Control behaviour of f2py with the use of command line switches, directives and signature files."
image:
    path: /assets/img/mandelbrot/mandelbrotgray-feature.jpg
tags: [python, fortran, hpc]
---
See part 1 [here](https://dexterritory.online/posts/spicing-up-your-python-code).

In the [previous part](https://dexterritory.online/posts/spicing-up-your-python-code) we explored how to wrap Fortran code in Python module and concluded with some unanswered questions. In the second part we are going to address some of them.

<!--more-->

## Prerequisities
Be sure to read the previous post in the series. We will use the same Fortran code for computing Mandelbrot image as before.

## Our goal for today
Ok, so you got your Fortran code working and you wrapped it nicely into a Python module. Everything is working fine except possibly few things:

- there are some Fortran functions that were exposed in your Python module but are not supposed to be a part of a public API.
- `f2py` guessed wrong your intent. Maybe the function it created returns an array when you wanted to pass your own array to be filled with data?

Even if the above use cases are not very common, kowing how to handle them can come in handy. For today's example we will use the same code as in `mandelbrot.f90` file in the first part. Recall that the module created by `f2py` had the following contents:

- a function `color(p_x, p_y, maxiter)` returning a color for given coordinates and maximum number of iterations.
- a function `compute_colors(top, left, bot, right, cols, rows, maxiter)` that computed colors for every point in specified grid and returned array with the result of the computation.

While this is perfectly fine one might consider the following changes:

- don't include `color` function in the wrapped module. It does no harm to include it for now but maybe you just don't want to expose it for some reason. Why? Well in some real-world scenario it might be a function with potentially dangerous side effects that you just don't want your clients to invoke by themselves or you just don't want to clutter your Python module.
- you want to pass an array to be filled to the `compute_colors` function rather than have `numpy` allocate and pass an array for you (it should be possible to do since the `compute_colors` function in `mandelbrot.f90` takes an output array as an argument). Why? Maybe you want to invoke the function multiple times and want to save cycles on allocations and deallocations of an array. In some more complex scenarios, the array you pass there can already hold some data and the algorithm you implemented in Fortran modifies only a part of the array. As you see there are some legitimate reasons to pass the output array yourself.

## Restricting exposed functions using command line switches
There are two useful command line options to use with `f2py` that allow one to restrict what functions should be exported in your Python module: `skip: ... :` and `only: ... :`. Both of them accept sequence of space-separated function names. So, to not include `color` function we can either run
~~~bash
f2py -m mandelbrot -c mandelbrot.f90 only: compute_colors :
~~~
or
~~~bash
f2py -m mandelbrot -c mandelbrot.f90 skip: color :
~~~
After that we can verify the content of our module by importing it and running `help`.
~~~python
import mandelbrot

help(mandelbrot)
~~~

This gives us
~~~text
NAME
    mandelbrot

DESCRIPTION
    This module 'mandelbrot' is auto-generated with f2py (version:2).
    Functions:
    Fortran 90/95 modules:
      mandelbrot --- compute_colors().
~~~
So everything worked as planned.

### Caveats of using only and skip
There are some mistakes you can make when using skip or only that can give you a headache. This is mainly because their peculiar format.

- If you want to specify multiple functions you need to separate them with space. Also, the final and starting space is important.
If you forget the starting space `f2py` will fail to parse your command. If you forget the final space the last function won't be taken into account. So to correctly export only `foo` and `bar` functions you need to use `only: foo bar :`.
- Be careful of typos! `f2py` will ignore nonexisting functions and won't throw an error or a warning at you. Of course, careful analysis of its output would reveal for what functions the wrapper has been created but it is easy to miss. As a consequence it is possible to create a module without a content e.g. when you specify only one function to export and mispell its name. What would happen then? If you expect `f2py` to behave in a civilized fashion you would be wrong. In such a case it simply creates module that will fail to import in a very inelegant way by segfaulting.

## Controlling arguments' intent using f2py directives
Intent of the arguments can be controlled by using `!f2py` directives. You basically repeat your Fortran's argument definition with possibly different intents. Possible values include `in`, `out`, `in,out` and `inplace` with obvious meaning.
You could modify your Fortran code like so to facilitate inplace computing of your Mandelbrot image.
~~~fortran
! File mandelbrot.f90
module mandelbrot
  implicit none
contains

  integer(8) function color(p_x, p_y, maxiter)
    real(8), intent(in) :: p_x, p_y
    integer(8), intent(in) :: maxiter
    real(8) :: x_tmp, y_tmp, x, y

    color = 0
    x = p_x
    y = p_y

    do while (x * x + y * y < 4 .and. color < maxiter)
       x_tmp = x * x - y * y + p_x
       y_tmp = 2 * x * y + p_y
       x = x_tmp
       y = y_tmp
       color = color + 1
    end do
  end function color

  subroutine compute_colors( &
       top, left, bot, right, & ! coordinates of corners
       out_arr, cols,rows, & ! grid size
       maxiter) ! maximum number of iteratoins
    implicit none;
    !f2py integer(8), dimension(rows, cols), intent(inplace) :: out_arr    
    real(8), intent(in) :: top, left, bot, right
    integer(8), intent(in) :: rows, cols
    integer(8), dimension(rows, cols), intent(out) :: out_arr
    integer(8), intent(in) :: maxiter

    integer(8) :: i, j
    real(8) :: x, y

    do i = 1, cols
       ! Compute abscissa of point corresponding to
       ! (i, j element in grid) - basically a convex
       ! combination.
       x =  left * (cols-i)/(cols-1.0) + (i-1)/(cols-1.0) * right
       do j = 1, rows
          ! As above but for the ordinate
          y = (rows-j)/(rows-1.0) * top + (j-1)/(rows-1.0) * bot
          out_arr(j, i) = color(x, y, maxiter)
       end do
    end do
  end subroutine compute_colors
end module mandelbrot
~~~
If you run `f2py` again and inspect generated function's signature you will see that it changed appropriately.
~~~text
out_arr = compute_colors(top,left,bot,right,out_arr,maxiter,[cols,rows])

Wrapper for ``compute_colors``.

Parameters
----------
top : input float
left : input float
bot : input float
right : input float
out_arr :  rank-2 array('q') with bounds (rows,cols)
maxiter : input long

Other Parameters
----------------
cols : input long, optional
    Default: shape(out_arr,1)
rows : input long, optional
    Default: shape(out_arr,0)

Returns
-------
out_arr : rank-2 array('q') with bounds (rows,cols)
~~~

Obviously you also have to modify the Python script from the previous part that uses your Fortran module. You could do it like so:
~~~python
from argparse import ArgumentParser
import imageio
import numpy as np
from mandelbrot import mandelbrot

def main():
    parser = ArgumentParser()
    parser.add_argument('--width', type=int, default=1920,
                        help='width of the image')
    parser.add_argument('--height', type=int, default=1080,
                        help='height of the image')
    parser.add_argument('-t', type=float, default=1,
                        help='ordinate of top-left corner')
    parser.add_argument('-l', type=float, default=-2,
                        help='abscissa of top-left corner')
    parser.add_argument('-b', type=float, default=-1,
                        help='ordinate of bottom-right corner')
    parser.add_argument('-r', type=float, default=1,
                        help='abscissa of bottom-right corner')
    parser.add_argument('-i', type=float, default=100,
                        help='maximum number of iterations')
    parser.add_argument('output', type=str, default=None,
                        help='path to the output file')

    args = parser.parse_args()
    arr = np.zeros((args.height, args.width), dtype='int64')
    mandelbrot.compute_colors(args.t, args.l, args.b, args.r, arr, maxiter=args.i)
    arr = arr / np.max(arr)
    imageio.imwrite(args.output, arr)

if __name__ == '__main__':
    main()
~~~

## Signature files
The signature files are by far the most powerful, although slightly verbose, way of controlling the behaviour of `f2py`. 
Using signature files boils down roughly to the following

1. Generate signature file using using `f2py` command line interface. This will produce a file describing every function in Fortran code that `f2py` was able to discover, along with its signature and arguments' intent.
2. Modify the signature file to your liking. Modifications you introduce are totally up to you and they include tweaking arguments' intent, deleting functions that should not be exported or fixing data types (if they were not detected correctly).
3. Generate Python module using your modified signature file.

We will now discuss all of the above steps.

### Generating signature file

The signature file can be generated using the `-h` command line switch like so:

~~~bash
f2py -m <module> -h <signature file> <source file>
~~~

where

- `<module>` is the name of the intended Python module that will wrap your Fortran code.
- `<signature file>` is a path (or at least a file name) of the signature file that will be written for you. It should have *pyf* extension.
- `<source file>` is a file with your Fortran code.

It is important to note that `f2py` will refuse to write a signature file if it already exists, unless you also specify `--overwrite-signature` switch. This is really cool. After all you will most likely make some adjustments to the signature file so you certainly don't want to accidentally lose your work.

After running the above command for the original `mandelbrot.f90`, you will get a signature file file that look like this

~~~fortran
!    -*- f90 -*-
! Note: the context of this file is case sensitive.

python module mandelbrot ! in 
    interface  ! in :mandelbrot
        module mandelbrot ! in :mandelbrot:mandelbrot.f90
            function color(p_x,p_y,maxiter) ! in :mandelbrot:mandelbrot.f90:mandelbrot
                real(kind=8) intent(in) :: p_x
                real(kind=8) intent(in) :: p_y
                integer(kind=8) intent(in) :: maxiter
                integer(kind=8) :: color
            end function color
            subroutine compute_colors(top,left,bot,right,out_arr,cols,rows,maxiter) ! in :mandelbrot:mandelbrot.f90:mandelbrot
                real(kind=8) intent(in) :: top
                real(kind=8) intent(in) :: left
                real(kind=8) intent(in) :: bot
                real(kind=8) intent(in) :: right
                integer(kind=8) dimension(rows,cols),intent(out),depend(rows,cols) :: out_arr
                integer(kind=8) intent(in) :: cols
                integer(kind=8) intent(in) :: rows
                integer(kind=8) intent(in) :: maxiter
            end subroutine compute_colors
        end module mandelbrot
    end interface 
end python module mandelbrot

! This file was auto-generated with f2py (version:2).
! See http://cens.ioc.ee/projects/f2py2e/
~~~

We can observe that the information in this signature files correspond perfectly to the functions that `f2py` generated automatically. Let's see if we can reproduce it to mimic the behaviour of command line switches and `!f2py` directives shown in the previous section.

### Modifying the signature file

As previously stated, the signature file contains definitions of all the functions that will be wrapped into Python module. Needless to say, if you delete any of those definitions the corresponding function won't be included in your module. Exposing only a specific subset of functions is therefore really simple, just don't include them in you signature and you are good to go. Modifying the intent of the arguments is also simple, we just need to modify the appropriate line describing given argument.

Taking the above remarks into account, the modified signature file for our module could look like this:

~~~fortran
python module mandelbrot ! in 
    interface  ! in :mandelbrot
        module mandelbrot ! in :mandelbrot:mandelbrot.f90
            subroutine compute_colors(top,left,bot,right,out_arr,cols,rows,maxiter) ! in :mandelbrot:mandelbrot.f90:mandelbrot
                real(kind=8) intent(in) :: top
                real(kind=8) intent(in) :: left
                real(kind=8) intent(in) :: bot
                real(kind=8) intent(in) :: right
                integer(kind=8) dimension(rows,cols),intent(inplace),depend(rows,cols) :: out_arr
                integer(kind=8) intent(in) :: cols
                integer(kind=8) intent(in) :: rows
                integer(kind=8) intent(in) :: maxiter
            end subroutine compute_colors
        end module mandelbrot
    end interface 
end python module mandelbrot
~~~

Now that we have the signature file prepared, we need to instruct `f2py` to use it when creating our module.


### Generating Python module using signature file

To generate Python module using the signature file you just need to specify it as one of the files to compile. Assuming your generated signature file is `mandelbrot.pyf` you would need to run the following command.

~~~bash
f2py -c mandelbrot.pyf mandelbrot.f90
~~~

That's it, your module is generated and ready to import.

## Signature files or directives and command line switches?

Seeing that we achieved the same result with signature files as with `f2py` directives and command line switches, a natural question is: which solution is the best?

The answer depends largely on your goals and preferences. Personally I prefer having a signature file, especially if the code is a part of some larger project. I think it clearly shows your intent(ions) and how you expect the code to be generated, like a sort of a blueprint of your wrapper module. Having said that, I sometimes go with command line switches and `f2py` directives, especially when I need a quick and dirty solution and don't have to care too much for maintainability or style.

## What's next?

In this post I demonstrated several ways to tweak `f2py` behaviour to restrict set of exported functions and modify arguments' intents. This, along with the first post, should allow you to wrap a broad class of Fortran codes into nice Python modules. Naturally the next step is to include such a wrapped code in an installable package, which is what we will focus on in the next part of this series.

## Sources and further reading
Obviously there is more to signature files and `!f2py` directives than I described - there is no point in repeating documentation. For the reference here are the docs you might want to consult should you need some information not contained in my posts.

- [Signature file documentation](https://docs.scipy.org/doc/numpy-1.15.0/f2py/signature-file.html)
- [More information on `f2py` bindings and directives](https://docs.scipy.org/doc/numpy-1.15.0/f2py/python-usage.html)
