---
title: Improving float array parser
date:  2021-05-16 16:40:02 +0530
classes: wide
excerpt: Improving C++ string to float array parser runtime from O(n^2) to linear
---

In a [previous post]({% link _posts/2021-05-03-twil.md %}), I had mentioned that the [current JSON parser in ArduPilot](https://github.com/ArduPilot/ardupilot/blob/3de3f5750108cd896f4aab4955097cd445d5cbdb/libraries/SITL/SIM_AirSim.cpp#L135) for the AirSim data can be improved. Well, this post is the result of testing and writing the new parser!

A brief recap, [this article](https://nee.lv/2021/02/28/How-I-cut-GTA-Online-loading-times-by-70/) brought to attention the quadratic runtime of `sscanf` for parsing (highly recommend reading). Looking at the JSON parser implementation, it was also using `sscanf`, which is particularly concerning for the float array and `vector3f` array part. Here's a simplified implementation of the Vector3f part:

```cpp
void origParser(const char* data, std::vector<float>& v) {
    const char* p = data;
    if (*p++ != '[') {
        printf("Missing opening bracket for array data\n");
        return;
    }

    int i = 0;

    while(true) {
        if (sscanf(p, "%f,%f,%f,", &v[i], &v[i+1], &v[i+2]) != 3) {
            printf("Failed to parse data: %s\n", p);
            return;
        }

        i += 3;

        // Goto 3rd occurence of ,
        p = strchr(p,',');
        if (!p) {
            return;
        }
        p++;

        p = strchr(p,',');
        if (!p) {
            return;
        }
        p++;

        p = strchr(p,',');
        if (!p) {
            return;
        }
        p++;

        // Reached end of point cloud
        if (p[0] == ']') {
            break;
        }
    }
}
```

It parses the string to match `"%f,%f,%f,"`, and then skips till the end, with some actually redundant error checking in between. The `sscanf` part is O(n^2) where n is the length of the input data, and then another O(n) parse for the `strchr`.

The new implementation:

```cpp
void newParser(const char* data, std::vector<float>& v) {
    const char* p = data;
    if (*p++ != '[') {
        printf("Missing opening bracket for array data\n");
        return;
    }

    int i = 0;
    char* end;

    while(*p) {
        v[i] = strtod(p, &end);
        // Skip comma after number
        p = ++end;
        if (*p == '\0')
            break;

        v[i+1] = strtod(p, &end);
        // Skip comma after number
        p = ++end;
        if (*p == '\0')
            break;

        v[i+2] = strtod(p, &end);

        // Truncated input or end of array
        if (*end == '\0' || *end == ']')
            break;

        i += 3;
        p = ++end;
    }
}
```

The new implementation uses [`strtod`](http://www.cplusplus.com/reference/cstdlib/strtod/), and its ability to set the endptr to the char after the parsed number, which removes the need to use `strchr` to get the comma. It's easier to read, plus much faster as can be seen from the timing measurements below:

```text
Num. of points 100
Time for original parser: 57 us
Time for new parser: 33 us

Num. of points 1000
Time for original parser: 585 us
Time for new parser: 303 us

Num. of points 10000
Time for original parser: 21409 us
Time for new parser: 3070 us

Num. of points 50000
Time for original parser: 484706 us
Time for new parser: 15972 us
```

The entire code can be found in [this gist](https://gist.github.com/rajat2004/03194708d9cfe83e84161dd84464fcc9).

However, I haven't yet decided to open a PR to make these changes. My main reason currently is that passing Lidar data (which this parser is used for) is limited to around 3200 points due to UDP limitations (~65kB). At this level, the difference is negligible, and doesn't really justify (at least to me) the code churn just for this. If I open a PR to add the [`airsim-non-gps` branch](https://github.com/rajat2004/ardupilot/tree/airsim-non-gps) though, then the parser improvement will definitely be a part of it.
