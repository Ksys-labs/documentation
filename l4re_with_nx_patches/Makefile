DOCFILES=instruction_eng.md instruction_rus.md gdb_fiasco.md
PDFS=$(patsubst %.md,%.pdf,$(DOCFILES))

.PHONY: $(PDFS)
all: $(PDFS)

$(PDFS): %.pdf: %.md
	gimli -file $< -stylesheet style.css

.PHONY: clean
clean:
	rm *.pdf
