+++
tags = []
categories = []
draft = false
image = ""
weight = 5
writer = "Lukas Herman"
title = "High-Performance String Concatenation in Go"
date = "2018-06-25T00:20:16-04:00"
+++

Go is a great and powerful programming language as well as my personal favorite. It's statically compiled, high performant, and great at concurrent computing. Even so, it's still the programmer's responsibility to utilize the programming language's strength fully. 

In this post, I'll talk about how to concatenate strings efficiently. As usual, we'll start with the naive approach and slowly improve upon it.

In every approach, I averaged **30 trials** for each *n*. Also, ```word```, the word we are concatenating, is always my name, "Lukas". If you're not interested in the details, you can skip to the end to see the charts. ? 

## Naive Way: Use '+' operator
The most straightforward and possibly intuitive way to concatenate strings is to use the '+' operator, but, this is not the best way.

https://github.com/lherman-cs/string-builder/blob/master/naive.go

```go
func concat(word string, n int) string {
	s := ""
	for i := 0; i < n; i++ {
		s += word
	}
	return s
}
```

The problem with this approach is that strings, in most languages including Go, are **immutable**. Meaning, everytime you concatenate strings, it will actually create a new string that contains "old_string" + "new_string" and then delete the "old_string".

![naive](/content/images/2018/06/naive.png)

*Disclaimer: This illustration is not an actual memory map. This functions to help to understand what's going on under the '+' operator*

As you can see from the picture above, this approach can be very expensive, especially when ```s``` is a very large string.


### *n* vs *time(s)*

|     n     |   Naive  |
|:---------:|:--------:|
|     10    | 6.702e-7 |
|    100    | 1.784e-5 |
|    1000   | 8.618e-4 |
|   10000   |   0.038  |
|   100000  |   3.658  |
|  1000000  |  317.979 |
|  10000000 |     -    |
| 100000000 |     -    |

*n = 10000000 and n = 100000000 are painfully slow. So, I skipped them...*

## Optimization #1: Use a string slice
The first thing that we can do to avoid that unnecessary copying intrinsic in the naive way, is to store the strings in a Go slice.

https://github.com/lherman-cs/string-builder/blob/master/opt1.go

```go
func concat(word string, n int) string {
	var sb []string

	for i := 0; i < n; i++ {
		sb = append(sb, word)
	}
	return strings.Join(sb, "")
}
```

![opt1-1](/content/images/2018/06/opt1-1.png)

This way, instead of repetitively copying and deleting, which wastes CPU cycles unnecessary, we basically append the new string to the array and join them at the end.

*Disclaimer: This illustration is not an actual memory map.*

### *n* vs *time(s)*

|     n     |   Naive  |   Opt 1  |
|:---------:|:--------:|:--------:|
|     10    | 6.702e-7 | 3.601e-6 |
|    100    | 1.784e-5 | 1.142e-5 |
|    1000   | 8.618e-4 | 8.172e-5 |
|   10000   |   0.038  | 7.857e-4 |
|   100000  |   3.658  |   0.007  |
|  1000000  |  317.979 |   0.127  |
|  10000000 |     -    |   1.153  |
| 100000000 |     -    |  12.574  |


## Optimization #2: Use a byte slice instead
In the previous approach, we had a string slice, here we are leveraging the byte slice instead. Since a character is a byte, the byte slice represents an array of characters.

Since strings are actually arrays of characters, using a string slice is still expensive, because it is a 2-D array, which is more complex and has more associated overhead than its 1-D representation. 

Another problem with the previous approach is 
```strings.Join(sb, "")```, because joining strings is quite expensive. To read more details about how this approach can optimize the code further, please check out [this article](https://medium.com/@felipedutratine/in-golang-should-i-work-with-bytes-or-strings-8bd1f5a7fd48). It's a great article!

The byte as opposed to string slice optimizes the Join function as well as takes us from 2-D to 1-D.

https://github.com/lherman-cs/string-builder/blob/master/opt2.go

```go 
func concat(word string, n int) string {
	var sb []byte

	for i := 0; i < n; i++ {
		sb = append(sb, word...)
	}
	return string(sb)
}
```

In ```sb = append(sb, word...)```, the syntax ```word...``` tells Go to unpack the string.

### *n* vs *time(s)*
|     n     |   Naive  |   Opt 1  |   Opt 2  |
|:---------:|:--------:|:--------:|:--------:|
|     10    | 6.702e-7 | 3.601e-6 | 9.489e-7 |
|    100    | 1.784e-5 | 1.142e-5 | 6.011e-6 |
|    1000   | 8.618e-4 | 8.172e-5 | 4.097e-5 |
|   10000   |   0.038  | 7.857e-4 | 3.439e-4 |
|   100000  |   3.658  |   0.007  |   0.001  |
|  1000000  |  317.979 |   0.127  |   0.007  |
|  10000000 |     -    |   1.153  |   0.072  |
| 100000000 |     -    |  12.574  |   0.639  |


## Optimization #3: Replace string with []byte

If you paid attention to the previous code, there's actually another small optimization we can make. We still wasted the CPU cycles to convert a string to []byte and the other way around. For more information about why this is slow, please check out [this article](https://syslog.ravelin.com/byte-vs-string-in-go-d645b67ca7ff).

https://github.com/lherman-cs/string-builder/blob/master/opt3.go

```go
func concat(word []byte, n int) []byte {
	var sb []byte

	for i := 0; i < n; i++ {
		sb = append(sb, word...)
	}
	return sb
}
```

*Note: Notice that now ```concat``` doesn't return string anymore. This might be not what you want to use in your application.*

### *n* vs *time(s)*

|     n     |   Naive  |   Opt 1  |   Opt 2  |   Opt 3  |
|:---------:|:--------:|:--------:|:--------:|:--------:|
|     10    | 6.702e-7 | 3.601e-6 | 9.489e-7 | 5.009e-7 |
|    100    | 1.784e-5 | 1.142e-5 | 6.011e-6 | 3.053e-6 |
|    1000   | 8.618e-4 | 8.172e-5 | 4.097e-5 | 5.042e-5 |
|   10000   |   0.038  | 7.857e-4 | 3.439e-4 | 2.919e-4 |
|   100000  |   3.658  |   0.007  |   0.001  |   0.001  |
|  1000000  |  317.979 |   0.127  |   0.007  |   0.007  |
|  10000000 |     -    |   1.153  |   0.072  |   0.063  |
| 100000000 |     -    |  12.574  |   0.639  |   0.584  |


# Charts
![Screenshot-from-2018-06-24-16-50-54](/content/images/2018/06/Screenshot-from-2018-06-24-16-50-54.png)

![Screenshot-from-2018-06-24-16-51-47](/content/images/2018/06/Screenshot-from-2018-06-24-16-51-47.png)

![Screenshot-from-2018-06-24-16-52-28](/content/images/2018/06/Screenshot-from-2018-06-24-16-52-28.png)


# Conclusion
In this article, we've improved the performance, when ```n = 1000000```, by **45425x**. So, whenever you do a lot of string concatenations, don't use '+' operator. Use ```[]byte``` instead of a ```string```. Also, if you notice, the Go standard library uses ```[]byte``` more often for its performance and flexibility! 

You can apply this technique to **any programming language**.

As always, you can always check out the full code [here](https://github.com/lherman-cs/string-builder)

---

# Extra
If you don't want to do the optimization yourself, Go has a standard library that does string concatenation pretty efficiently. You can go to [this link](https://golang.org/pkg/strings/#Builder) for more details.

```go
func concat(word []byte, n int) string {
	var sb strings.Builder

	for i := 0; i < n; i++ {
		sb.Write(word)
	}
	return sb.String()
}
```

Here we compare optimization #3 to the standard library, ```strings.Builder```.

|     n     |   Opt 3  |    Std   |
|:---------:|:--------:|:--------:|
|     10    | 5.009e-7 | 8.862e-7 |
|    100    | 3.053e-6 | 7.074e-6 |
|    1000   | 5.042e-5 | 4.974e-5 |
|   10000   | 2.919e-4 | 5.033e-4 |
|   100000  |   0.001  |   0.002  |
|  1000000  |   0.007  |   0.012  |
|  10000000 |   0.063  |   0.111  |
| 100000000 |   0.584  |   1.070  |

![Screenshot-from-2018-06-24-16-59-55](/content/images/2018/06/Screenshot-from-2018-06-24-16-59-55.png)

For ```n=100000000``` our optimization #3 is nearly **2x** as fast as the Go standard library version. 

*Note: This may be because the standard library must be more generalized, which causes overheads.*