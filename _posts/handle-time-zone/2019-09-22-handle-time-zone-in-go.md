---
layout: post
title:  "Handling time zones in Go"
date:   2019-09-22 09:29:20 +0200
categories: go
---


Many beginner developers get confused in dealing with timezones. This article explains -  
  

 - How to store them in DB?
 - How to parse them in Go?

  
When timezone is stored in the DB, always stick to one standard timezone, the ideal would be to save UTC time and while presenting them, convert it to various timezones as per requirement.  
  
I am taking MySQL as an example of storing time, but the below solution is DB agnostic. As per MySQL documentation, There are two ways one can store time in MySQL.

- DATETIME — The DATETIME type is used for values that contain both date and time parts. MySQL retrieves and displays DATETIME values in `YYYY-MM-DD hh:mm:ss` format. The supported range is `1000-01-01 00:00:00` to `9999-12-31 23:59:59`.
- TIMESTAMP — The TIMESTAMP datatype is used for values that contain both date and time parts. TIMESTAMP has a range of `1970-01-01 00:00:01` UTC to `2038-01-19 03:14:07` UTC.

In this article, I will use DATETIME for example.

Now, the other and most important thing is to read and convert it into any timezone.

Below is an example of how we can achieve this in Go. Let’s first define a map from country to [IANA identifiers](https://www.iana.org/time-zone) and some utility functions.

{% highlight go %}
package main

import (
	"fmt"
	"errors"
	"time"
)

type Country string


const (
	Germany Country = "Germany"
	UnitedStates Country  = "United States"
	NewZealand Country = "New Zealand"
)

// timeZoneID is a map of Country to its IANA standard timezone identifier
var timeZoneID = map[Country]string{
	Germany:      "Europe/Berlin",
	UnitedStates: "America/Los_Angeles",
	NewZealand:   "Pacific/Auckland",
}

// TimeZoneID returns a IANA identifier for a given Country.
func (c Country) TimeZoneID() (string, error) {
	if id, ok := timeZoneID[c]; ok {
		return id, nil
	}
	return "", errors.New("invalid country")
}

// TimeIn returns time in timezone tz with fmt format
func TimeIn(t time.Time, tz, fmt string) (string, error) {
	
	// https:/golang.org/pkg/time/#LoadLocation loads location on
	// the basis of
	loc, err := time.LoadLocation(tz)
	if err != nil {
		return "", err
	}
	
	// convert current time to specific location, e.g Germany in given format
	return t.In(loc).Format(fmt), nil
}

func main() {
	// Get the timezone
	tz, err := UnitedStates.TimeZoneID()
	if err != nil {
		//handle error
	}

	usTime, err := TimeIn(time.Now(), tz, time.RFC3339)
	if err != nil {
		// handle error
	}

	fmt.Printf("Time in %s: %s",
		UnitedStates,
		usTime,
	)
}
{% endhighlight %}

You can play with the full example in the [Go playground](https://play.golang.org/p/8yI0EexvVk5)

#### Caveat:

As per go documentation of Load location, The time zone database needed by LoadLocation may
not be present on all systems, especially non-Unix systems. LoadLocation looks 
in the directory or uncompressed zip file named by the `ZONEINFO` environment variable,
 if any, then looks in known installation locations on Unix systems, and finally looks in `$GOROOT/lib/time/zoneinfo.zip`.
 
#### With Docker:
 
 By default, it comes with Go installation. but in case you deploy using Docker and build multi-stage docker Alpine image. You can add line given below.
 
    RUN apk add tzdata
 
 This will add timezone information into /usr/share/timezone in the alpine image.
 
 Also, do not forget to set the Environment variable ZONEINFO to /usr/share/timezone.
 
    ZONEINFO=/usr/share/timezone
 
 Just for reference, Below is the sample Dockerfile.

{% highlight docker %}
FROM golang:1.12-alpine AS build_base
RUN apk add --update bash make git
WORKDIR /yourapp
ENV GO111MODULE on
COPY go.mod .
COPY go.sum .
RUN go mod download

FROM build_base AS server_builder
COPY . ./
RUN GOOS=linux GOARCH=amd64 go build -ldflags="-w -s" -o /go/bin/yourapp

# Build final image
FROM alpine
# To add tzdata (Time zone data) to the image.
RUN apk add tzdata
{% endhighlight %}