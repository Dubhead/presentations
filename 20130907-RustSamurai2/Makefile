# Makefile

DESTDIR=out

${DESTDIR}/rust-condition.html: rust-condition.md
	@mkdir -p ${DESTDIR}
	pandoc --self-contained -s -t slidy -s $< -o $@

# eof
