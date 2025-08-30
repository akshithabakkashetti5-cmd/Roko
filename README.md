import os


if uploaded_files:
all_chunks = []
with st.spinner("Reading and chunking PDFs..."):
for f in uploaded_files:
pdf_bytes = f.read()
pages = extract_text_with_pages(pdf_bytes)
for page_num, text in pages:
for ch in sentence_chunk(text, max_chars=max_chars, overlap=overlap):
all_chunks.append({
"text": ch,
"page": page_num,
"source": f.name,
})
if not all_chunks:
st.warning("No text found in the uploaded PDFs. Are they scanned images? Try OCR first.")
else:
index, _ = build_faiss_index(all_chunks)
st.session_state.chunks = all_chunks
st.session_state.index = index
st.success(f"Indexed {len(all_chunks)} chunks from {len(uploaded_files)} PDF(s).")


question = st.text_input("Ask a question about your PDFs:", placeholder="e.g., What are the key differences between FDMA and TDMA?")


col1, col2 = st.columns([2, 1])
with col1:
if st.button("Answer", disabled=not (question and st.session_state.index)):
with st.spinner("Retrieving and generating answer..."):
contexts = retrieve(question, st.session_state.chunks, st.session_state.index, k=top_k)
if not contexts:
st.error("I couldn't find relevant content in the PDFs.")
else:
answer = generate_answer(question, contexts)
st.markdown("### 🧩 Answer")
st.write(answer)


# Build readable citations panel
st.markdown("### 📌 Sources")
for c in contexts:
snippet = (c["text"][:350] + "...") if len(c["text"]) > 350 else c["text"]
st.markdown(
f"**{c['source']} — p. {c['page']}**\n\n> {snippet}"
)


# Export
txt = f"Question: {question}\n\nAnswer:\n{answer}\n\nSources:\n" + "\n".join(
[f"- {c['source']} p.{c['page']} (score={c['score']:.3f})" for c in contexts]
)
st.download_button("Download answer (.txt)", txt, file_name="answer.txt")


with col2:
st.markdown("### ℹ️ Notes")
st.markdown(
"- This app does **not** see the internet; it only uses your PDFs.\n"
"- If your PDFs are scans, use an OCR tool (e.g., Adobe, Tesseract) first.\n"
"- For better answers, upload fewer but **high‑quality** notes or narrow to a single chapter.\n"
)


st.markdown("---")
with st.expander("Advanced: Prompt used (for debugging)"):
if st.session_state.get("chunks") and question:
ctx = retrieve(question, st.session_state.chunks, st.session_state.index, k=min(2, top_k))
st.code(build_prompt(question, ctx))
