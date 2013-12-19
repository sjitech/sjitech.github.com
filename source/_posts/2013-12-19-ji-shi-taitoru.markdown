---
layout: post
title: "Node.js vs C vs Java vs Python"
date: 2013-12-19 10:12
comments: true
categories: 
description: 
keywords: 
---

Node.js、C、Java、Pythonの比較は複雑ですが、とりあえず、純粋な言語性能を計ってみました。後でもっと現実に近いケースで計りましょう。   
今回のケースでは、Node.js = 0.85 C = 0.83 Java = 100+ Pythonぐらい。  
  
厳密に言うと、今回はNode.jsの性能ではなくV8 Javascriptのです。  
  
環境: Mac OS X 10.9, Intel Core i7 CPU 4 Core  
テスト内容：オブジェクト割当、配列割当、素数計算  
実行時状況：  
　　CPU：皆 0% (たまにCは100%)  
　　スレッド:  
　　　　Node.js 4個, Cは1個。Javaは17個。  
　　　　※ javascriptは単スレッドですが、それを管理するために別スレッドがあります、数はCPU数とは関係なくほぼ4固定です。  
　　　　JavaはCより早いのは、自動的に並列されるかもしれません？  
  
限定ケースの結果ですのでいろいろ訳あり。  
例えば、C、Pythonはオプティマイズされていない。  

ソース：
Node.js  
<pre><code>
/** This code was based on Google I/O 2012:
Breaking the JavaScript Speed Limit with V8
(http://www.youtube.com/watch?v=UJPdhx5zTaw) **/

function Primes() {
	this.prime_count = 0;
	this.primes = new Array(50000);
};

Primes.prototype.getPrimeCount = function() { return this.prime_count; }
Primes.prototype.getPrime = function(i) { return this.primes[i]; }
Primes.prototype.addPrime = function(i) { this.primes[this.prime_count++] = i; }
Primes.prototype.isPrimeDivisible = function(candidate) {
		for (var i = 1; i &lt; this.prime_count; ++i) {
			if ((candidate % this.primes[i]) == 0) return true;
		}
		return false;
	}


function main() {
	p = new Primes();
	var c = 1;
	var st = Date.now();
	while (p.getPrimeCount() &lt; 50000) {
		if (!p.isPrimeDivisible(c)) {
			p.addPrime(c);
		}
		c++;
	}
	console.log((Date.now()-st));
	console.log(p.getPrime(p.getPrimeCount()-1));
}

main();
</code></pre>
  
Java  
<pre><code>
import static java.lang.System.out;
import java.util.Date;

public class primes {

	public static class Primes
	{
		public int prime_count;
		public int[] primes = new int[50000];

		public int getPrimeCount () { return this.prime_count; }
		public int getPrime (int i) { return this.primes[i]; }
		public void addPrime (int p) { this.primes[this.prime_count++] = p; }

		public boolean isPrimeDivisible(int candidate) {
			for (int i = 1; i &lt; this.prime_count; ++i) {
				if ((candidate % this.primes[i]) == 0) return true;
			}
			return false;
		}
	}
	
	public static void main(String[] args) {
		Primes p = new Primes();
		int c = 1;
		Date st = new Date();
		while (p.getPrimeCount() &lt; 50000) {
			if (!p.isPrimeDivisible(c)) {
				p.addPrime(c);
			}
			c++;
		}
		out.println(new Date().getTime()-st.getTime());
		out.println(p.getPrime(p.getPrimeCount() - 1));
	}

}
</code></pre>
  
C  
<pre><code>
/** This code was based on Google I/O 2012: 
Breaking the JavaScript Speed Limit with V8 
(http://www.youtube.com/watch?v=UJPdhx5zTaw) **/

#include &lt;stdio.h&gt;
#include &lt;sys/time.h&gt;
#include &lt;sys/types.h&gt;

static int64_t microSecondOfNow() {
    struct timeval t;
    gettimeofday(&t, NULL);
    return ((int64_t) t.tv_sec) * (1000 * 1000) + t.tv_usec;
}

class Primes {
	public:
		int getPrimeCount() const { return prime_count; }
		int getPrime(int i) const { return primes[i]; }
		void addPrime(int i) { primes[prime_count++] = i; }
		
		bool isPrimeDivisible(int candidate) {
			for (int i = 1; i &lt; prime_count; ++i) {
				if ((candidate % primes[i]) == 0) return true;
			}
			return false;
		}
	private:
		volatile int prime_count;
		volatile int primes[50000];
};

int main() {
	Primes p;
	int c = 1;
	int64_t st = microSecondOfNow();
	while (p.getPrimeCount() &lt; 50000) {
		if (!p.isPrimeDivisible(c)) {
			p.addPrime(c);
		}
		c++;
	}
	printf("%lld\n", (microSecondOfNow()-st)/1000);
	printf("%d\n", p.getPrime(p.getPrimeCount() - 1));
}
</code></pre>
  
Python  
<pre><code>
import datetime

class Primes:
    def __init__(self):
        self.prime_count = 0
        self.primes = [0]*50000

    def getPrimeCount(self):
        return self.prime_count

    def getPrime(self, i):
        return self.primes[i]

    def addPrime(self, i):
        self.primes[self.prime_count] = i
        self.prime_count += 1

    def isPrimeDivisible(self, candidate):
        for i in range(1, self.prime_count - 1):
            if (candidate % self.primes[i]) == 0:
                return True
        return False

p = Primes()
c = 1
st = datetime.datetime.now()
while p.getPrimeCount() &lt; 50000:
    if not p.isPrimeDivisible(c):
        p.addPrime(c)
    c += 1
print(datetime.datetime.now()-st);
print(p.getPrime(p.getPrimeCount()-1))

</code></pre>

