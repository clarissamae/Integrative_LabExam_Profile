
<script setup>
  import { reactive, ref } from "vue"
  import { formatDistanceToNow } from 'date-fns'
  import { toast } from "vue3-toastify"
  import { auth, db } from "@/config/firebaseConfig"
  import { onAuthStateChanged } from "firebase/auth"
  import { collection, doc, getDoc, updateDoc, query, where, orderBy, onSnapshot, getDocs } from "firebase/firestore"

  /* Profile data */
  const profile = reactive({
    name: "Guest User",
    course: "",
    location: "",
    about: "",
    major: "",
    year: "",
    email: "",
    interests: "",
    skills: "",
  });

  const isModalOpen = ref(false)
  const editForm = reactive({ ...profile })

  const recentPosts = ref([])


  const fetchUserPosts = (userId) => {
    const postsQuery = query(
      collection(db, 'posts'),
      where('uid', '==', userId),
      orderBy('createdAt', 'desc')
    )
    onSnapshot(postsQuery, (snapshot) => {
      recentPosts.value = snapshot.docs.map(docSnap => ({
        id: docSnap.id,
        ...docSnap.data()
      }))
    })
  }


  const openModal = () => {
    Object.assign(editForm, profile)
    isModalOpen.value = true
  }
  const closeModal = () => {
    isModalOpen.value = false
  }

  /* Save profile and update Firestore */
  const saveProfile = async () => {
    try {
      const user = auth.currentUser
      if (!user) throw new Error("No logged-in user found")

      await updateDoc(doc(db, "users", user.uid), {
        firstName: editForm.name.split(" ")[0] || "",
        lastName: editForm.name.split(" ")[1] || "",
        program: editForm.course,
        location: editForm.location,
        about: editForm.about,
        major: editForm.major,
        year: editForm.year,
        interests: editForm.interests,
        skills: editForm.skills,
      })

      Object.assign(profile, editForm)
      closeModal()
      toast.success("Profile updated successfully!")
    } catch (error) {
      console.error("Error updating profile:", error)
      toast.error("Failed to update profile. Try again!")
    }
  }

  /* Fetch logged-in user data and posts */
  onAuthStateChanged(auth, async (user) => {
    if (user) {
      profile.email = user.email || ""
      fetchUserPosts(user.uid)

      try {
        const docRef = doc(db, "users", user.uid)
        const docSnap = await getDoc(docRef)
        if (docSnap.exists()) {
          const data = docSnap.data()
          profile.name =
            data.firstName && data.lastName
              ? `${data.firstName} ${data.lastName}`
              : "User"
          profile.course = data.program || ""
          profile.location = data.location || ""
          profile.about = data.about || "Hi! I am new here"
          profile.major = data.major || ""
          profile.year = data.year || ""
          profile.interests = data.interests || ""
          profile.skills = data.skills || ""
        }
      } catch (error) {
        console.error("Error fetching user data:", error)
      }
    } else {
      profile.name = "Guest User"
      profile.course = ""
      recentPosts.value = []
    }
  })
</script>


<template>

  <div class="bg-gray-100 min-h-screen">
    <div class="bg-indigo-700 relative overflow-hidden">
      <img
        src="https://images.unsplash.com/photo-1504384308090-c894fdcc538d?auto=format&fit=crop&w=1200&q=80"
        alt="Cover"
        class="w-full h-40 object-cover opacity-50"
      />

      <!-- Profile Info -->
      <div class="absolute inset-0 flex items-end px-6 pb-4">
        <img
          src="https://images.unsplash.com/photo-1529626455594-4ff0802cfb7e?auto=format&fit=facearea&facepad=3&w=256&h=256&q=80"
          alt="Profile"
          class="w-20 h-20 rounded-full border-4 border-white shadow-md"
        />
        <div class="ml-4 text-white">
          <h1 class="text-xl font-bold">{{ profile.name }}</h1>
          <p class="text-sm">{{ profile.course }}</p>
        </div>
      </div>
    </div>

    <!-- ✅ Details Section -->
    <div class="bg-white shadow p-6 flex items-center justify-between">
      <p class="text-gray-700">📍 {{ profile.location }}</p>
      <button
        @click="openModal"
        class="bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700"
      >
        Edit Profile
      </button>
    </div>

    <!-- ✅ About Section -->
    <div class="mt-6 bg-white rounded-lg p-6">
      <h2 class="text-lg font-semibold text-gray-800 mb-4">About</h2>
      <p class="text-gray-600">{{ profile.about }}</p>

      <div class="mt-4 grid grid-cols-1 sm:grid-cols-2 gap-4 text-sm">
        <p><span class="font-semibold">Major:</span> {{ profile.major }}</p>
        <p><span class="font-semibold">Year:</span> {{ profile.year }}</p>
        <p><span class="font-semibold">Email:</span> {{ profile.email }}</p>
        <p><span class="font-semibold">Interests:</span> {{ profile.interests }}</p>
        <p><span class="font-semibold">Skills:</span> {{ profile.skills }}</p>
      </div>
    </div>


    <!-- ✅ Recent Posts Section -->
    <div class="mt-6 bg-white rounded-lg p-6">
      <h2 class="text-lg font-semibold text-gray-800 mb-4">Recent Posts</h2>
      <div v-if="recentPosts.length === 0" class="text-gray-500">No posts yet</div>
      <div class="space-y-4">
        <div
          v-for="post in recentPosts"
          :key="post.id"
          class="p-4 border border-gray-100 rounded-lg">
          
          <div class="flex justify-between items-center">
            <h3 class="font-semibold text-gray-800">{{ profile.name }}</h3>
            <span class="text-sm text-gray-500">
              {{
                post.createdAt?.toDate
                ? formatDistanceToNow(post.createdAt.toDate(), { addSuffix: true })
                : 'just now'
              }}
            </span>
          </div>
          <p class="mt-2 text-gray-700">{{ post.content }}</p>
        </div>
      </div>
    </div>

    <!-- ✅ Friends Section -->
    <div class="bg-white shadow rounded-lg mt-6 p-4">
      <h2 class="text-lg font-semibold mb-4 flex items-center border-b pb-2">
        <i class="fas fa-user-friends mr-2 text-blue-500"></i> Friends · 
        <span class="ml-1 text-gray-500 text-sm">120</span>
      </h2>

      <div class="grid grid-cols-3 sm:grid-cols-4 gap-4">
        <!-- Friend 1 -->
        <div class="flex flex-col items-center text-center hover:bg-gray-50 rounded-lg p-2 cursor-pointer">
          <img
            class="h-20 w-20 rounded-lg object-cover mb-2"
            src="https://images.unsplash.com/photo-1494790108377-be9c29b29330?auto=format&fit=facearea&facepad=2&w=256&h=256&q=80"
            alt="Jane Doe"
          />
          <h3 class="text-sm font-semibold text-gray-900">Jane Doe</h3>
        </div>

        <!-- Friend 2 -->
        <div class="flex flex-col items-center text-center hover:bg-gray-50 rounded-lg p-2 cursor-pointer">
          <img
            class="h-20 w-20 rounded-lg object-cover mb-2"
            src="https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?auto=format&fit=facearea&facepad=2&w=256&h=256&q=80"
            alt="John Smith"
          />
          <h3 class="text-sm font-semibold text-gray-900">John Smith</h3>
        </div>

        <!-- Friend 3 -->
        <div class="flex flex-col items-center text-center hover:bg-gray-50 rounded-lg p-2 cursor-pointer">
          <img
            class="h-20 w-20 rounded-lg object-cover mb-2"
            src="https://images.unsplash.com/photo-1517841905240-472988babdf9?auto=format&fit=facearea&facepad=2&w=256&h=256&q=80"
            alt="Emily Chen"
          />
          <h3 class="text-sm font-semibold text-gray-900">Emily Chen</h3>
        </div>

        <!-- Friend 4 -->
        <div class="flex flex-col items-center text-center hover:bg-gray-50 rounded-lg p-2 cursor-pointer">
          <img
            class="h-20 w-20 rounded-lg object-cover mb-2"
            src="https://images.unsplash.com/photo-1558898479-33c0057a5d12?auto=format&fit=facearea&facepad=2&w=256&h=256&q=80"
            alt="Sophia Lee"
          />
          <h3 class="text-sm font-semibold text-gray-900">Sophia Lee</h3>
        </div>
      </div>

      <div class="mt-4 text-center">
        <button class="text-sm text-blue-500 hover:underline">See all friends</button>
      </div>
    </div>
  </div>



  <!-- ✅ Edit Profile Modal -->
  <div v-if="isModalOpen" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
    <div class="bg-white rounded-xl shadow-xl w-full max-w-lg relative flex flex-col max-h-[90vh]">
    
      <button
        type="button"
        @click="closeModal"
        class="absolute top-4 left-4 flex items-center space-x-1 px-3 py-1 rounded-full border border-gray-300 bg-white text-gray-700 hover:bg-gray-100 shadow-sm z-50"
        >
        <!-- Arrow icon -->
        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
        </svg>
        <span>Back</span>
      </button>

      <h2 class="text-2xl font-bold mb-6 text-center text-gray-800 pt-4">Edit Profile</h2>

      <!-- Scrollable Form Area -->
      <div class="overflow-y-auto px-6 pb-6 flex-1">
        <form @submit.prevent="saveProfile" class="space-y-5">
          <div>
            <label class="block text-sm font-medium text-gray-700">Full Name</label>
            <input
              v-model="editForm.name"
              type="text"
              placeholder="Enter your full name"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Course</label>
            <input
              v-model="editForm.course"
              type="text"
              placeholder="Enter your course"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Location</label>
            <input
              v-model="editForm.location"
              type="text"
              placeholder="Enter your location"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Email</label>
            <input
              v-model="editForm.email"
              type="email"
              placeholder="Enter your email"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">About</label>
            <textarea
              v-model="editForm.about"
              rows="3"
              placeholder="Tell something about yourself"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none resize-none">
            </textarea>
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Major</label>
            <input
              v-model="editForm.major"
              type="text"
              placeholder="Enter your major"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Year</label>
            <input
              v-model="editForm.year"
              type="text"
              placeholder="Enter your year"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Interests</label>
            <input
              v-model="editForm.interests"
              type="text"
              placeholder="Your interests"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div>
            <label class="block text-sm font-medium text-gray-700">Skills</label>
            <input
              v-model="editForm.skills"
              type="text"
              placeholder="Your skills"
              class="mt-1 block w-full border border-gray-300 rounded-lg px-4 py-2 shadow-sm focus:ring-2 focus:ring-indigo-400 focus:outline-none"
            />
          </div>

          <div class="flex justify-end mt-4">
            <button
              type="submit"
              class="px-6 py-2 rounded-lg bg-indigo-600 text-white font-medium hover:bg-indigo-700 shadow-md transition-colors duration-200"
              >
              Submit
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>  
</template>
