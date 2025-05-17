# Flash-cards-by-mixed-subjects

from flask import Flask, request, jsonify
from datetime import datetime
import re
import random
from collections import defaultdict

app = Flask(__name__)

# In-memory storage for flashcards (in a real system, use a database)
flashcards = []

# Subject classification rules
SUBJECT_KEYWORDS = {
    "Physics": ["newton", "force", "mass", "acceleration", "velocity", "energy", "quantum", "thermodynamics"],
    "Chemistry": ["atom", "molecule", "reaction", "chemical", "periodic table", "bond", "organic"],
    "Biology": ["cell", "photosynthesis", "dna", "organism", "evolution", "ecosystem", "mitochondria"],
    "Mathematics": ["equation", "algebra", "calculus", "derivative", "integral", "matrix", "probability"],
    "History": ["war", "century", "empire", "revolution", "ancient", "medieval", "colonial"],
    "Computer Science": ["programming", "algorithm", "database", "python", "java", "binary", "compiler"]
}

def classify_subject(text):
    text = text.lower()
    subject_scores = {subject: 0 for subject in SUBJECT_KEYWORDS}
    
    for subject, keywords in SUBJECT_KEYWORDS.items():
        for keyword in keywords:
            if re.search(r'\b' + re.escape(keyword) + r'\b', text):
                subject_scores[subject] += 1
                
    # Get the subject with the highest score
    best_subject = max(subject_scores.items(), key=lambda x: x[1])[0]
    return best_subject if subject_scores[best_subject] > 0 else "General"

@app.route('/flashcard', methods=['POST'])
def add_flashcard():
    data = request.json
    
    # Validate input
    if not all(key in data for key in ['student_id', 'question', 'answer']):
        return jsonify({"error": "Missing required fields"}), 400
    
    # Classify the subject
    question = data['question']
    answer = data['answer']
    combined_text = f"{question} {answer}"
    subject = classify_subject(combined_text)
    
    # Create and store the flashcard
    flashcard = {
        "student_id": data['student_id'],
        "question": question,
        "answer": answer,
        "subject": subject,
        "created_at": datetime.now().isoformat()
    }
    flashcards.append(flashcard)
    
    return jsonify({
        "message": "Flashcard added successfully",
        "subject": subject
    }), 201

@app.route('/get-subject', methods=['GET'])
def get_subject_flashcards():
    student_id = request.args.get('student_id')
    limit = int(request.args.get('limit', 5))  # Default to 5 if not specified
    
    # Filter flashcards by student_id
    student_flashcards = [fc for fc in flashcards if fc['student_id'] == student_id]
    
    if not student_flashcards:
        return jsonify({"error": "No flashcards found for this student"}), 404
    
    # Group flashcards by subject
    subject_groups = defaultdict(list)
    for fc in student_flashcards:
        subject_groups[fc['subject']].append(fc)
    
    # Shuffle the flashcards within each subject group
    for subject in subject_groups:
        random.shuffle(subject_groups[subject])
    
    # Create a mixed batch with at most one flashcard per subject
    mixed_batch = []
    subjects = list(subject_groups.keys())
    random.shuffle(subjects)  # Shuffle the order of subjects
    
    # Add one flashcard from each subject until we reach the limit
    for subject in subjects:
        if len(mixed_batch) >= limit:
            break
        if subject_groups[subject]:  # If there are flashcards for this subject
            mixed_batch.append(subject_groups[subject].pop())
    
    # If we haven't reached the limit, add random flashcards from any subject
    while len(mixed_batch) < limit and student_flashcards:
        # Flatten all remaining flashcards
        remaining_flashcards = [fc for subj in subject_groups.values() for fc in subj]
        if not remaining_flashcards:
            break
        mixed_batch.append(remaining_flashcards.pop())
    
    # Prepare the response (excluding student_id and created_at)
    response = [
        {
            "question": fc["question"],
            "answer": fc["answer"],
            "subject": fc["subject"]
        }
        for fc in mixed_batch
    ]
    
    return jsonify(response)

if __name__ == '__main__':
    app.run(debug=True)
