package main

import (
	"fmt"
	"regexp"
)

func main() {
	var bs = []byte{0x26, 0xff, 0x01, 0x00, 0x00, 0x00, 0x40, 0x40, 0x41, 0x41, 0x28}
	var s = "\\01\\x00+\\x40\\x40.\\x28"
	matched, err := regexp.Match(s, bs)
	fmt.Println(matched)
	fmt.Println(err)
}

bytes.Index