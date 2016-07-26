# FTL (Flash Translation Layer)

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/).

## On the necessity of having an FTL

An additional component is required to hide the inner characteristics of NAND flash memory and expose only an array of LBAs to the host. This component is called the **Flash Translation Layer (FTL)**, and resides in the SSD controller.

## Logical block mapping

- translates logical block addresses (LBAs) from the host space into physical block addresses (PBAs) in the physical NAND-flash memory space
- Page-level mapping
- Block-level mapping
- Log-block mapping
