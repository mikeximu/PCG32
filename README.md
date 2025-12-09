# PCG32
快速高性能随机数

go
```

import (
	"crypto/rand"
	"encoding/binary"
	"fmt"
)

type PCG32 struct {
	state uint64
	inc   uint64
}

func NewPCG32(seed, seq uint64) *PCG32 {
	return &PCG32{
		state: seed,
		inc:   (seq << 1) | 1,
	}
}

func (p *PCG32) Next() uint32 {
	oldstate := p.state
	p.state = oldstate*6364136223846793005 + p.inc
	xorshifted := uint32(((oldstate >> 18) ^ oldstate) >> 27)
	rot := uint32(oldstate >> 59)
	return (xorshifted >> rot) | (xorshifted << ((-rot) & 31))
}

func NewPCG32Prod() *PCG32 {
	var b [16]byte
	rand.Read(b[:])                                   // 必须使用系统硬件随机源，确保高熵不可预测
	state := binary.LittleEndian.Uint64(b[0:8])       // 初始状态
	stream := binary.LittleEndian.Uint64(b[8:16]) | 1 // stream 必须为奇数，否则随机流退化

	return NewPCG32(state, stream)
}

// 无偏随机，高性能 v := PCGRange(rng, 301)  // 均匀 0~300
func PCGRange(p *PCG32, n uint32) uint32 {
	threshold := (^uint32(0) - n + 1) % n // 避免偏差
	for {
		x := p.Next()
		if x >= threshold {
			return x % n
		}
	}
}
func main() {
	rng := NewPCG32Prod()

	for i := 0; i < 10; i++ {
		// 生成 0~255 (2^8 - 1) ;2的幂次方速度最快
		fmt.Println(rng.Next() & 255)
	}
}
```

rust
```
use rand::rngs::OsRng;
use rand::RngCore;

pub struct PCG32 {
    state: u64,
    inc: u64, // stream id (must be odd)
}

impl PCG32 {
    pub fn new_prod() -> Self {
        let mut seed = [0u8; 16];
        OsRng.fill_bytes(&mut seed); // 高熵安全随机源
        
        let state = u64::from_le_bytes(seed[0..8].try_into().unwrap());
        let mut inc  = u64::from_le_bytes(seed[8..16].try_into().unwrap());
        inc = (inc << 1) | 1; // 必须为奇数，PCG 序列要求

        Self { state, inc }
    }

    #[inline]
    pub fn next_u32(&mut self) -> u32 {
        let oldstate = self.state;
        self.state = oldstate
            .wrapping_mul(6364136223846793005)
            .wrapping_add(self.inc);
        let xorshifted = (((oldstate >> 18) ^ oldstate) >> 27) as u32;
        let rot = (oldstate >> 59) as u32;
        xorshifted.rotate_right(rot)
    }
}
```
