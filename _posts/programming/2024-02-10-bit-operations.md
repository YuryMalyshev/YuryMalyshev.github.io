---
layout: post
title:  "Pointers, Bit operations, and Modbus (Part 1)"
date:   2024-02-10 19:00:00 +0200
excerpt_separator: <!--more-->
categories: programming
---

### How did I get here

As I was re-writing an older implementation of the Modbus server protocol that I inherited from the previous guys at the company, I realized we don't utilize all the standard functionality. Particularly, we don't use "coils". 

<!--more-->

For those who aren't familiar with the Modbus protocol - there are two main datatypes: coils/bits and registers/words (16-bit values). Of course, it's possible to store larger values as a multiple of registers and divide a single register into smaller sections. This is exactly how we handle flags - there are a few registers that store up to 16 flags each.

There are some obvious downsides when it comes to handling flags if they're treated as registers since we always read/write 16 flags at once. For example, if I want to set bit 5, then I should first get the current data, modify that single bit, and then write the whole register back. However, with the "write coils" command, I could set the bit with a single write command.

### Data structure

One of the goals of this project was to preserve the registers. The readout of flags bit by bit should be optional. To make it possible, we can use pointers.

Let's imagine that we have 8 registers. Registers 1 and 3 contain flags. We can re-create this with the following code:

{% highlight C %}
uint16_t regs[8] = {0,1,2,3,4,5,6,7}.
uint16_t *coils[2] = {&regs[1], &regs[3]};
{% endhighlight %}

Here, I create an array of pointers to the original data. So, if we change the values in the regs array, the values will also change in the coils array and vice versa. 

### Reading the data

Now that we have the data, we can implement the reading function. Read multiple coils command takes two parameters: the start address and the number of bits to read. Therefore, in essence, we have to perform the following operations on our data:

1. Bit shift to the right by the start address.
2. Apply the mask to the bytes.
3. Trim the data to the least number of bytes.

However, in reality, it's a bit more complicated, and we can apply some optimization. Here's the function I came up with:

{% highlight C %}
int getBits(int start, int number, unsigned char* buf)
{
    // Calculate all needed values
    int first = start>>4;           // index of the 1st relevant register
    int startMask = start&0x0F;     // 1 reg = 16 bits. Move in 1 register intervals
    int total = ((startMask+number-1)>>4)+1; // total of relevant registers 1...N
    int maxBytes = total*2;         // bytes needed for the calculations. 1 uint16_t = 2 bytes
    int shift = startMask&0x07;     // 1 reg = 16 bits = number 16. N % 16 == N & 0x0F
    int offset = startMask>>3;      // bit shifting byte offset. 8 bits is 1 byte
    int bytes = ((number-1)>>3)+1;  // number of byes 1..N
    
    // set the memory to 0
    memset(buf, 0, maxBytes);
    
    // copy data from the relevant registers to the buffer
    for(int i = first, j = 0; i < first+total; i++, j++)
        memcpy(&buf[j*2], coils[i], 2);
    
    // apply the bit shift
    for(int i=offset, j=0; i < maxBytes; i++, j++)
    {
        unsigned char newval = (buf[i] >> shift);
        if(i < maxBytes-1)
            newval |= (buf[i+1] << (8-shift));
        buf[j] = newval;
    }
    
    // apply the mask to last byte
    if((number&0x07) != 0)
        buf[bytes-1] &= ((1 << (number&0x07))-1);
    
    // return number of relevant bytes
    return bytes;
}
{% endhighlight %}

Let's look at it more closely. The first thing we need to know, we can't use the data in the coils directly since we use pointers. If I try to cast the coils array to the byte array, I'll end up accessing all the values from the registers array; that's why we need to copy the data from the coils array to an intermediate buffer. That's where the first optimization can come into play - we don't need to copy everything to the buffer; it's enough to copy the registers we will be working with.

After that, we can apply the bit shift. Since the original data was 16-bit but the buffer is 8-bit, we need to start shifting from the right index. For example, if the start address mask is 8, then we don't need to do a bit shift at all. 

Finally, the output data should only contain the relevant bits, and the rest should be zeros. I accomplish that by applying a mask to the last byte.  

<hr><br>
In the next part, we'll look at how to set the bits.
