# 1337ify
C program to generate leet speak wordlists from bare strings

Takes strings from STDIN and converts them to leet speak for all permutations using a mapping file.

Save the following source as `1337ify.c` and compile `gcc -O3 -o 1337ify 1337ify.c`<br />
```c
// benchmark:
// echo "some_really_long_string" | 1337ify 1337ify.map.simple.txt | pv --line-mode --rate > /dev/null

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <ctype.h>
#include <stdint.h>

#define MAXINPSTR 1001 // ASCIIZ
#define MAXSUBLEN 4
#define MAXMAPLEN (MAXSUBLEN + 3) // ASCIIZ
#define MAXOUTSTR ((MAXINPSTR - 1) * MAXSUBLEN) + 1 // ASCIIZ

typedef struct xn
{
   uint8_t sub[MAXSUBLEN];
   int32_t len;
   void *next;
} xlat_node;

int32_t addnode(uint8_t *p);
int32_t transform(uint8_t *instr, int32_t inlen);
int32_t increment(xlat_node **pxn, uint8_t *instr, int32_t inlen);
void usage();
void license();

xlat_node *xlat_table[256] = {[0 ... 255] = NULL};

int main(int argc, char **argv)
{
   FILE *fp;
   uint8_t instr[MAXINPSTR];
   int32_t i, j = 0, inlen;
   xlat_node *px, *py;

   if(argc != 2)
   {
      usage();

      return -1;
   }
   if((fp = fopen(argv[1], "r")) == NULL)
   {
      fprintf(stderr, "Failed to open <mapping file>.\n");

      return -1;
   }
   while(!feof(fp))
   {
      if(fgets(instr, MAXINPSTR, fp))
      {
         for(i = 0; i < strlen(instr); i++)
         {
            if(instr[i] < ' ') instr[i] = 0;
         }
         j++;
         if(addnode(instr))
         {
            fclose(fp);
            fprintf(stderr, "Line number %d.\n", j);

            return -1;
         }
      }
   }
   fclose(fp);
   for(i = 0; i < 256; i++)
   {
      if((px = xlat_table[i]) != NULL)
      {
         py = px;
         while(py->next) py = py->next;
         py->next = px;
      }
   }
   while(1)
   {
      if(fgets(instr, MAXINPSTR, stdin) != NULL)
      {
         for(i = 0; i < strlen(instr); i++)
         {
            if(instr[i] < ' ') instr[i] = 0;
            instr[i] = tolower(instr[i]);
         }
         inlen = strlen(instr);
         if(transform(instr, inlen))
         {
            fprintf(stderr, "Transformation error.\n", j);

            return -1;
         }
      }
      else
      {
         if(feof(stdin)) break;
      }
   }

   return 0;
}

void usage()
{
   fprintf(stderr, "1337ify <mapping file>\n\n");
   fprintf(stderr, "Reads strings from STDIN and permutes them to \"leet speak\" based on\n");
   fprintf(stderr, "substitutions from a <mapping file>.  All input is converted to lower\n");
   fprintf(stderr, "case prior to substitution being performed.  The <mapping file> uses\n");
   fprintf(stderr, "line-based associations in the following example format:\n");
   fprintf(stderr, "a=a\na=A\na=@\na=/-\\\nb=b\nb=B\nb=|3\nw=w\nw=\\/\\/\n");
   fprintf(stderr, "Blank lines and lines with unsupported syntax will abort the program.\n");
   fprintf(stderr, "Substitution sequences may not exceed %d characters. You should map\n", MAXSUBLEN);
   fprintf(stderr, "a character to itself if you wish to preserve the original character.\n");
   fprintf(stderr, "Unmapped characters will not be altered.  Input strings will be truncated\n");
   fprintf(stderr, "to %d characters.\n\n", (MAXINPSTR - 1));
   license(); 
}

void license()
{
	char *license = "\
 Released under the terms of the BSD 3-Clause \"BSD Modified\" license.\n\
\n\
 Copyright (c) 2020, Bill Chaison\n\
 All rights reserved.\n\
\n\
 Redistribution and use in source and binary forms, with or without \n\
 modification, are permitted provided that the following conditions \n\
 are met:\n\
\n\
 1. Redistributions of source code must retain the above copyright notice,\n\
    this list of conditions and the following disclaimer.\n\
 2. Redistributions in binary form must reproduce the above copyright notice,\n\
    this list of conditions and the following disclaimer in the documentation\n\
    and/or other materials provided with the distribution.\n\
 3. Neither the name of the copyright holder nor the names of its\n\
    contributors may be used to endorse or promote products derived from this\n\
    software without specific prior written permission.\n\
\n\
 THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS \"AS IS\"\n\
 AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE\n\
 IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE\n\
 ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE\n\
 LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR\n\
 CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF\n\
 SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS\n\
 INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN\n\
 CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)\n\
 ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE\n\
 POSSIBILITY OF SUCH DAMAGE.";

	fprintf(stderr, "[License]\n%s\n", license);
	fprintf(stderr, "\n");
}

int32_t addnode(uint8_t *p)
{
   uint8_t idx, sub[MAXSUBLEN];
   int32_t len, i, j;
   xlat_node *node, *pxn;

   if(strlen(p) >= MAXMAPLEN)
   {
      fprintf(stderr, "Mapping file line too long at: (%s)\n", p);

      return -1;
   }
   if(*(p + 1) != '=')
   {
      fprintf(stderr, "Mapping file syntax error at: (%s)\n", p);

      return -1;
   }
   len = strlen(p) - 2;
   idx = *p;
   memcpy(sub, p + 2, len);
   if(xlat_table[idx] == NULL)
   {
      if((node = malloc(sizeof(xlat_node))) == NULL)
      {
         fprintf(stderr, "Node allocation failure.\n", p);

         return -1;
      }
      memcpy(node->sub, sub, len);
      node->len = len;
      node->next = NULL;
      xlat_table[idx] = node;
   }
   else
   {
      pxn = xlat_table[idx];
      while(pxn)
      {
         if(pxn->len == len)
         {
            j = 0;
            for(i = 0; i < len; i++)
            {
               if(pxn->sub[i] == sub[i]) j++;
            }
            if(j == len)
            {
               pxn = NULL;
               break;
            }
         }
         if(pxn->next != NULL)
         {
            pxn = pxn->next;
         }
         else
         {
            break;
         }
      }
      if(pxn)
      {
         if((pxn->next = malloc(sizeof(xlat_node))) == NULL)
         {
            fprintf(stderr, "Node allocation failure.\n", p);

            return -1;
         }
         pxn = pxn->next;
         memcpy(pxn->sub, sub, len);
         pxn->len = len;
         pxn->next = NULL;
      }
   }

   return 0;
}

int32_t transform(uint8_t *instr, int32_t inlen)
{
   xlat_node *current[MAXINPSTR];
   int32_t i, j = 0, k;
   uint8_t outstr[MAXOUTSTR];

   for(i = 0; i < inlen; i++)
   {
      current[i] = xlat_table[*(instr + i)];
      if(current[i]) j++;
   }
   if(!j)
   {
      printf("%s\n", instr);

      return 0;
   }
   while(1)
   {
      j = 0;
      for(i = 0; i < inlen; i++)
      {
         if(!current[i])
         {
            outstr[j++] = *(instr + i);
            outstr[j] = 0;
         }
         else
         {
            for(k = 0; k < current[i]->len; k++)
            {
               outstr[j + k] = current[i]->sub[k];
            }
            j += k;
            outstr[j] = 0;
         }
      }
      printf("%s\n", outstr);
      if(increment(current, instr, inlen) == inlen) break;
   }

   return 0;
}

int32_t increment(xlat_node **pxn, uint8_t *instr, int32_t inlen)
{
   int32_t i;

   for(i = 0; ; i++)
   {
      if(i == inlen)
      {
         break;
      }
      else
      {
         if(pxn[i] == NULL)
         {
            // just increment
         }
         else
         {
            if(pxn[i]->next == pxn[i])
            {
               // just increment
            }
            else
            {
               pxn[i] = pxn[i]->next;
               if(pxn[i] == xlat_table[*(instr + i)])
               {
                  // just increment
               }
               else
               {
                  break;
               }
            }
         }
      }
   }

   return i;
}
```

Example simple mapping file `1337ify.map.simple.txt`<br />
```
a=a
a=A
a=@
b=b
b=B
c=c
c=C
d=d
d=D
e=e
e=E
e=3
f=f
f=F
g=g
g=G
h=h
h=H
i=i
i=I
i=1
j=j
j=J
k=k
k=K
l=l
l=L
m=m
m=M
n=n
n=N
o=o
o=O
o=0
p=p
p=P
q=q
q=Q
r=r
r=R
s=s
s=S
s=5
s=$
t=t
t=T
u=u
u=U
v=v
v=V
w=w
w=W
x=x
x=X
y=y
y=Y
z=z
z=Z
```

Example complex mapping file `1337ify.map.complex.txt`<br />
```
a=a
a=A
a=@
a=4
a=/-\
b=b
b=B
b=8
b=|3
c=c
c=C
c=(
d=d
d=D
d=|)
e=e
e=E
e=3
f=f
f=F
f=|=
g=g
g=G
g=6
h=h
h=H
h=|-|
i=i
i=I
i=1
i=!
i=|
j=j
j=J
j=]
k=k
k=K
k=|<
l=l
l=L
l=1
l=!
l=|
l=|_
m=m
m=M
m=|\/|
m=/|\
n=n
n=N
n=|\|
o=o
o=O
o=0
o=*
o=()
p=p
p=P
q=q
q=Q
q=(,)
r=r
r=R
s=s
s=S
s=5
s=$
t=t
t=T
t=+
t=7
u=u
u=U
u=|_|
v=v
v=V
v=\/
w=w
w=W
w=\/\/
x=x
x=X
x=%
x=><
x=}{
y=y
y=Y
y=`/
z=z
z=Z
z=2
```
