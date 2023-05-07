---
toc: true
layout: post
comments: true
description: "Solutions to the exercises from the book The Go Programming Language: First Part"
categories: [go, exercise]
title: "Solutions to the exercises from the book The Go Programming Language (First Part)"
---

In this article, we will review solutions to the exercises presented in the first part of the book “The Go Programming Language” by Alan A.A. Donovan and Brain W. Kernighan. In an effort to refresh my knowledge, I have re-read this book and decided to complete all the exercises provided in it. I am delighted to share my solutions and receive feedback and comments on my code from other users. If you have any suggestions or improvements to my code, I would greatly appreciate your feedback.

*Exercise 1.1: Modify the echo program to also print os.Args[0], the name of the command that invoked it.*

~~~~go
package main

import (
 "fmt"
 "os"
)

func main() {
 for _, s := range os.Args {
  fmt.Println(s)
 }
}
~~~~

*Exercise 1.2: Modify the echo program to print the index and value of each of its arguments, one per line.*

~~~~go
package main

import (
   "fmt"
   "os"
)

func main() {
   for i, s := range os.Args {
      fmt.Println(i, " - ", s)
   }
}
~~~~

* Exercise 1.3: Experiment to measure the difference in running time between our potentially inefficient versions and the one that uses strings.Join.*

~~~~go
package main

import (
 "fmt"
 "os"
 "strings"
 "time"
)

func main() {

 start := time.Now()

 var s, sep string
 for i := 1; i < len(os.Args); i++ {
  s += sep + os.Args[i]
  sep = " "
 }
 fmt.Println(s)

 fmt.Println("First time: ", time.Since(start).Microseconds(), "ms")

 start1 := time.Now()

 s1, sep1 := "", ""
 for _, arg := range os.Args[1:] {
  s1 += sep1 + arg
  sep = " "
 }
 fmt.Println(s)

 fmt.Println("Second time: ", time.Since(start1).Microseconds(), "ms")

 start2 := time.Now()

 fmt.Println(strings.Join(os.Args[1:], " "))

 fmt.Println("First time: ", time.Since(start2).Microseconds(), "ms")

}
~~~~

*Exercise 1.4: Modify dup2 to print the names of all files in which each duplicated line occurs.*

~~~~go
package main

import (
   "bufio"
   "fmt"
   "os"
)

func main() {
   counts := make(map[string]int)
   filesName := make(map[string][]string)
   files := os.Args[1:]
   if len(files) == 0 {
      countLines(os.Stdin, counts, filesName)
   } else {
      for _, arg := range files {
         f, err := os.Open(arg)
         if err != nil {
            fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
            continue
         }
         countLines(f, counts, filesName)
         f.Close()
      }
   }
   for line, n := range counts {
      if n > 1 {
         fmt.Printf("%d\t%s\n", n, line)
         for _, j := range filesName[line] {
            fmt.Println(j)
         }
      }
   }
}

func countLines(f *os.File, counts map[string]int, filesName map[string][]string) {
   input := bufio.NewScanner(f)
   for input.Scan() {
      counts[input.Text()]++
      filesName[input.Text()] = append(filesName[input.Text()], f.Name())
   }
   // NOTE: ignoring potential errors from input.Err()
}
~~~~

*Exercise 1.5: Change the Lissajous program’s color palette to green on black, for added authenticity. To create the web color #RRGGBB, use color.RGBA{0xRR, 0xGG, 0xBB, 0xff}, where each pair of hexadecimal digits represents the intensity of the red, green, or blue component of the pixel.*

~~~~go
package main

import (
   "image"
   "image/color"
   "image/gif"
   "io"
   "log"
   "math"
   "math/rand"
   "net/http"
   "os"
   "time"
)

// var palette = []color.Color{color.White, color.Black}
var palette = []color.Color{color.RGBA{
   R: 0,
   G: 0,
   B: 0,
   A: 255,
}, color.RGBA{
   R: 0,
   G: 255,
   B: 0,
   A: 255,
}}

const (
   whiteIndex = 0 // first color in palette
   blackIndex = 1 // next color in palette
)

func main() {
   //!-main
   // The sequence of images is deterministic unless we seed
   // the pseudo-random number generator using the current time.
   // Thanks to Randall McPherson for pointing out the omission.
   rand.Seed(time.Now().UTC().UnixNano())

   if len(os.Args) > 1 && os.Args[1] == "web" {
      //!+http
      handler := func(w http.ResponseWriter, r *http.Request) {
         lissajousgreen(w)
      }
      http.HandleFunc("/", handler)
      //!-http
      log.Fatal(http.ListenAndServe("localhost:8000", nil))
      return
   }
   //!+main
   lissajousgreen(os.Stdout)
}

func lissajousgreen(out io.Writer) {
   const (
      cycles  = 5     // number of complete x oscillator revolutions
      res     = 0.001 // angular resolution
      size    = 100   // image canvas covers [-size..+size]
      nframes = 64    // number of animation frames
      delay   = 8     // delay between frames in 10ms units
   )
   freq := rand.Float64() * 3.0 // relative frequency of y oscillator
   anim := gif.GIF{LoopCount: nframes}
   phase := 0.0 // phase difference
   for i := 0; i < nframes; i++ {
      rect := image.Rect(0, 0, 2*size+1, 2*size+1)
      img := image.NewPaletted(rect, palette)
      for t := 0.0; t < cycles*2*math.Pi; t += res {
         x := math.Sin(t)
         y := math.Sin(t*freq + phase)
         img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
            blackIndex)
      }
      phase += 0.1
      anim.Delay = append(anim.Delay, delay)
      anim.Image = append(anim.Image, img)
   }
   gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
~~~~

*Exercise 1.6: Modify the Lissajous program to produce images in multiple colors by adding more values to palette and then displaying them by changing the third argument of SetColorIndex in some interesting way.*

~~~~go
package main

import (
   "image"
   "image/color"
   "image/gif"
   "io"
   "log"
   "math"
   "math/rand"
   "net/http"
   "os"
   "time"
)

var palette = []color.Color{
   color.White,
   color.RGBA{R: 255, A: 255},
   color.RGBA{G: 255, A: 255},
   color.RGBA{B: 255, A: 255},
   color.Black,
}

const (
   whiteIndex = 0 // first color in palette
   blackIndex = 1 // next color in palette
)

func main() {
   //!-main
   // The sequence of images is deterministic unless we seed
   // the pseudo-random number generator using the current time.
   // Thanks to Randall McPherson for pointing out the omission.
   rand.Seed(time.Now().UTC().UnixNano())

   if len(os.Args) > 1 && os.Args[1] == "web" {
      //!+http
      handler := func(w http.ResponseWriter, r *http.Request) {
         lissajous(w)
      }
      http.HandleFunc("/", handler)
      //!-http
      log.Fatal(http.ListenAndServe("localhost:8000", nil))
      return
   }
   //!+main
   lissajous(os.Stdout)
}

func lissajous(out io.Writer) {
   const (
      cycles  = 5     // number of complete x oscillator revolutions
      res     = 0.001 // angular resolution
      size    = 100   // image canvas covers [-size..+size]
      nframes = 64    // number of animation frames
      delay   = 8     // delay between frames in 10ms units
   )
   freq := rand.Float64() * 3.0 // relative frequency of y oscillator
   anim := gif.GIF{LoopCount: nframes}
   phase := 0.0 // phase difference
   for i := 0; i < nframes; i++ {
      rect := image.Rect(0, 0, 2*size+1, 2*size+1)
      img := image.NewPaletted(rect, palette)
      for t := 0.0; t < cycles*2*math.Pi; t += res {
         x := math.Sin(t)
         y := math.Sin(t*freq + phase)

         tmpIndex := uint8(i % len(palette))

         img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
            tmpIndex)
      }
      phase += 0.1
      anim.Delay = append(anim.Delay, delay)
      anim.Image = append(anim.Image, img)
   }
   gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
~~~~

*Exercise 1.7: The function call io.Copy(dst, src) reads from src and writes to dst. Use it instead of ioutil.ReadAll to copy the response body to os.Stdout without requiring a buffer large enough to hold the entire stream. Be sure to check the error result of io.Copy.*

~~~~go
package main

import (
 "fmt"
 "io"
 "log"
 "net/http"
 "os"
)

func main() {
 for _, url := range os.Args[1:] {
  resp, err := http.Get(url)
  if err != nil {
   fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
   os.Exit(1)
  }

  _, err = io.Copy(os.Stdout, resp.Body)
  if err != nil {
   log.Fatal(err)
   b, err := io.ReadAll(resp.Body)
   if err != nil {
    fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
    os.Exit(1)
   }
   fmt.Printf("%s", b)
  }
  resp.Body.Close()

 }
}
~~~~

*Exercise 1.8: Modify fetch to add the prefix http:// to each argument URL if it is missing. You might want to use strings.HasPrefix*

~~~~go
package main

import (
 "fmt"
 "io"
 "log"
 "net/http"
 "os"
 "strings"
)

func main() {
 for _, url := range os.Args[1:] {
  if !strings.HasPrefix(url, "http://") {
   url = "http://" + url
  }
  resp, err := http.Get(url)
  if err != nil {
   fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
   os.Exit(1)
  }
  _, err = io.Copy(os.Stdout, resp.Body)
  if err != nil {
   log.Fatal(err)
   b, err := io.ReadAll(resp.Body)
   if err != nil {
    fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
    os.Exit(1)
   }
   fmt.Printf("%s", b)
  }
  resp.Body.Close()
 }
}
~~~~

*Exercise 1.9: Modify fetch to also print the HTTP status code, found in resp.Status.*

~~~~go
package main

import (
 "fmt"
 "net/http"
 "os"
 "strings"
)

func main() {
 for _, url := range os.Args[1:] {
  if !strings.HasPrefix(url, "http://") {
   url = "http://" + url
  }
  resp, err := http.Get(url)
  if err != nil {
   fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
   os.Exit(1)
  }
  defer resp.Body.Close()

  fmt.Println("STATUS CODE", resp.StatusCode)
 }
}
~~~~

*Exercise 1.10: Find a web site that produces a large amount of data. Investigate caching by running fetchall twice in succession to see whether the reported time changes much. Do you get the same content each time? Modify fetchall to print its output to a file so it can be examined.*

~~~~go
package main

import (
 "fmt"
 "io"
 "log"
 "net/http"
 "os"
 "strings"
 "time"
)

func main() {
 start := time.Now()
 ch := make(chan string)
 for _, url := range os.Args[1:] {
  go fetch(url, ch) // start a goroutine
  go fetch(url, ch)
 }
 for range os.Args[1:] {
  fmt.Println(<-ch) // receive from channel ch
  fmt.Println(<-ch) // receive from channel ch
 }
 fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
 start := time.Now()
 resp, err := http.Get(url)
 if err != nil {
  ch <- fmt.Sprint(err) // send to channel ch
  return
 }

 f, err := os.Create(strings.Split(url, ".")[1] + ".txt")
 if err != nil {
  log.Println("Can't create file", err)
 }
 defer f.Close()

 nbytes, err := io.Copy(f, resp.Body)
 resp.Body.Close() // don't leak resources
 if err != nil {
  ch <- fmt.Sprintf("while reading %s: %v", url, err)
  return
 }
 secs := time.Since(start).Seconds()
 ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}
~~~~

*Exercise 1.11: Try fetchall with longer argument lists, such as samples from the top million web sites available at alexa.com. How does the program behave if a web site just doesn’t respond?*

The information I needed was not available on alexa.com, so I downloaded a file from kaggle.com.

~~~~go
package main

import (
 "encoding/csv"
 "fmt"
 "io"
 "log"
 "net/http"
 "os"
 "strings"
 "time"
)

func readCSVFile(filePath string) [][]string {
 f, err := os.Open(filePath)
 if err != nil {
  log.Fatal("Unable to read input file "+filePath, err)
 }
 defer f.Close()

 csvReader := csv.NewReader(f)
 records, err := csvReader.ReadAll()

 if err != nil {
  log.Fatal("Unable to parse file as CSV for "+filePath, err)
 }

 return records
}

func fetch(url string, ch chan<- string) {
 start := time.Now()
 resp, err := http.Get(url)
 if err != nil {
  ch <- fmt.Sprint(err) // send to channel ch
  return
 }

 nbytes, err := io.Copy(io.Discard, resp.Body)
 resp.Body.Close() // don't leak resources
 if err != nil {
  ch <- fmt.Sprintf("while reading %s: %v", url, err)
  return
 }
 secs := time.Since(start).Seconds()
 ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}

func main() {
 start := time.Now()
 ch := make(chan string)
 records := readCSVFile("top-1m.csv")
 url := ""
 for _, numberURL := range records {
  if !strings.HasPrefix(numberURL[1], "http://") {
   url = "http://" + numberURL[1]
  } else {
   url = numberURL[1]
  }
  go fetch(url, ch)
 }

 for range records {
  fmt.Println(<-ch)
 }

 fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())

}
~~~~

*Exercise 1.12: Modify the Lissajous server to read parameter values from the URL. For example, you might arrange it so that a URL like http://localhost:8000/?cycles=20 sets the number of cycles to 20 instead of the default 5. Use the strconv.Atoi function to convert the string parameter into an integer. You can see its documentation with go doc strconv.Atoi.*

~~~~go
package main

import (
   "image"
   "image/color"
   "image/gif"
   "io"
   "log"
   "math"
   "math/rand"
   "net/http"
   "net/url"
   "strconv"
)

var palette = []color.Color{
   color.White,
   color.RGBA{R: 255, A: 255},
   color.RGBA{G: 255, A: 255},
   color.RGBA{B: 255, A: 255},
   color.Black,
}

const (
   whiteIndex = 0 // first color in palette
   blackIndex = 1 // next color in palette
)

func main() {
   handler := func(w http.ResponseWriter, r *http.Request) {
      lissajous(w, r)
   }
   http.HandleFunc("/", handler)
   log.Fatal(http.ListenAndServe("localhost:8070", nil))
}

func lissajous(out io.Writer, r *http.Request) {
   const (
      res     = 0.001 // angular resolution
      size    = 100   // image canvas covers [-size..+size]
      nframes = 64    // number of animation frames
      delay   = 8     // delay between frames in 10ms units
   )
   var cycles = 5 // number of complete x oscillator revolutions
   if len(r.Header["Referer"]) > 0 {
      u, err := url.Parse(r.Header["Referer"][0])
      if err != nil {
         panic(err)
      }
      cyclesStr, _ := url.ParseQuery(u.RawQuery)
      if cyclesStr.Get("cycles") != "" {
         cycles, _ = strconv.Atoi(cyclesStr.Get("cycles"))
      }
   }

   freq := rand.Float64() * 3.0 // relative frequency of y oscillator
   anim := gif.GIF{LoopCount: nframes}
   phase := 0.0 // phase difference
   for i := 0; i < nframes; i++ {
      rect := image.Rect(0, 0, 2*size+1, 2*size+1)
      img := image.NewPaletted(rect, palette)
      for t := 0.0; t < float64(cycles)*2*math.Pi; t += res {
         x := math.Sin(t)
         y := math.Sin(t*freq + phase)
         img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
            blackIndex)
      }
      phase += 0.1
      anim.Delay = append(anim.Delay, delay)
      anim.Image = append(anim.Image, img)
   }
   gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
~~~~
