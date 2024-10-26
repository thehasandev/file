import { useGet, usePost } from "@bfirst/api-client";
import { ColorPicker } from "@bfirst/components-color-picker";
import { Icon } from "@bfirst/components-icon";
import { HCF } from "@bfirst/components-layout";
import { MultiselectSearch } from "@bfirst/components-multiselect-search";
import { TinymceEditor } from "@bfirst/components-tinymce-editor";
import { Button, CardBody, Checkbox, Input, Radio, Textarea } from "@bfirst/material-tailwind";
import { getImageUrl } from "@bfirst/utilities";
import { useReducer, useState } from "react";
import { useForm } from "react-hook-form";
import { useNavigate } from "react-router-dom";
import EmbedLink from "./EmbedLink";
import EmbedRelatedNews from "./EmbedRelatedNews";
import MediaBrowser from "./MediaBrowser";

export type Inputs = {
  shoulder?: string;
  headline: string;
  altheadline?: string;
  standfirst: string;
  imageCaption?: string;
};

export type StoryInputs = {
  title: string;
  meta: {
    featured_image: string;
    shoulder?: string;
    imageCaption?: string;
    altheadline?: string;
    intro: string;
  };
  content: string;
};

export interface StoryFormProps {
  onSubmit: (inputs: StoryInputs) => void;
  isError: boolean;
  loading: boolean;
  defaultData?: any;
  btnLabel: string;
}

export interface StateInterface {
  dialogOpen: boolean;
  openFrom: string;
}

const initialState: StateInterface = {
  dialogOpen: false,
  openFrom: "",
};

const reducer = function (curState: StateInterface, action: { type: string; payload?: string | boolean }) {
  switch (action.type) {
    case "setDialogOpen":
      return { ...curState, dialogOpen: (action.payload as boolean) || !curState.dialogOpen };
    case "setOpenFrom":
      return { ...curState, openFrom: action.payload as string };
    default:
      return { ...curState };
  }
};

export function StoryForm({ btnLabel, onSubmit, loading, isError, defaultData }: StoryFormProps) {
  const [state, dispatch] = useReducer(reducer, initialState);

  const [shoulderColor, setShoulderColor] = useState(defaultData?.story?.meta?.shoulder_color || "#5F5FB7");
  const [shoulderBlink, setShoulderBlink] = useState(defaultData?.story?.meta?.shoulder_blink || false);
  const [isOpenEmbed, setIsOpenEmbed] = useState(false);
  const [isOpenEmbedLink, setIsOpenEmbedLink] = useState(false);
  const [error, setError] = useState({
    authors: "",
    tags: "",
    categories: "",
    featuredImage: "",
    featuredVideo: "",
  });
  const [body, setBody] = useState(defaultData?.story.content || "");
  const [featuredImgUrl, setFeaturedImgUrl] = useState(defaultData?.story.meta.featured_image || "");
  const [moreImages, setMoreImages] = useState(defaultData?.story.meta.more_images || []);
  const [featuredVideo, setFeaturedVideo] = useState(defaultData?.story.meta.featured_video || "");
  const [search, setSearch] = useState({ authors: "", tags: "", categories: "" });
  const [selectedAuthors, setSelectedAuthors] = useState([]);
  const [selectedTags, setSelectedTags] = useState([]);
  const [selectedCategories, setSelectedCategories] = useState([]);
  const [featuredElement, setFeaturedElement] = useState<"image" | "video">(
    defaultData?.story.meta.featured_element || "image"
  );

  const { requestAsync: tagRequestAsync } = usePost(`api/v1/tags`);
  const { data: authorsData } = useGet(`api/v1/authors?name=${search.authors}`);
  const { data: tagsData } = useGet(`api/v1/tags?name=${search.tags}`);
  const { data: categoriesData } = useGet(`api/v1/categories?name=${search.categories}`);
  const navigate = useNavigate();
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<Inputs>();

  const handleAddTag = async (searchValue: string) => {
    const { data } = await tagRequestAsync({ name: searchValue });
    return data.data;
  };

  let debounce: string | number | NodeJS.Timeout | undefined;
  const debounceSearch = function (callback: () => void) {
    clearTimeout(debounce);
    debounce = setTimeout(() => {
      callback();
    }, 500);
  };

  const handleSelectFeaturedElement = function (e: any) {
    setFeaturedElement(e.target.value);
    setError((cur) => ({ ...cur, featuredImage: "", featuredVideo: "" }));
  };

  const onValidate = function (data: Inputs) {
    if (featuredElement === "video" && !featuredVideo)
      return setError((cur) => ({ ...cur, featuredVideo: "Featured Video is required" }));
    if (!featuredImgUrl) return setError((cur) => ({ ...cur, featuredImage: "Featured Image is required" }));
    if (!selectedAuthors.length) return setError((cur) => ({ ...cur, authors: "Author is required" }));
    if (!selectedTags.length) return setError((cur) => ({ ...cur, tags: "Tag is required" }));
    if (!selectedCategories.length) return setError((cur) => ({ ...cur, categories: "Category is required" }));

    const story = {
      title: data.headline,
      meta: {
        featured_element: featuredElement,
        featured_image: featuredImgUrl,
        featured_video: featuredVideo,
        shoulder: data.shoulder,
        shoulder_color: shoulderColor,
        shoulder_blink: shoulderBlink,
        imageCaption: data.imageCaption || defaultData?.story.meta.imageCaption,
        altheadline: data.altheadline,
        intro: data.standfirst,
        more_images: moreImages,
      },
      authors: selectedAuthors.map((author) => (author as { id: number }).id),
      tags: selectedTags.map((tag) => (tag as { id: number }).id),
      categories: selectedCategories.map((category) => (category as { id: number }).id),
      content: body,
    };

    onSubmit(story);
  };

  return (
    <form onSubmit={handleSubmit(onValidate)} className="h-full">
      <HCF>
        <HCF.Content>
          <CardBody className="flex flex-col gap-4">
            {/* ========== shoulder ========= */}
            <Input defaultValue={defaultData?.story.meta.shoulder} {...register("shoulder")} label="Shoulder" />

            {/* =========== shoulder color picker ============ */}
            <ColorPicker title="Shoulder Color" color={shoulderColor} onColorChange={setShoulderColor} />

            {/* ============ shoulder blink toggle ============ */}
            <div className="flex items-center gap-x-1 mb-5">
              <Checkbox
                checked={shoulderBlink}
                id="shoulder-blink-checkbox"
                onChange={(e) => setShoulderBlink(e.target.checked)}
              />
              <label htmlFor="shoulder-blink-checkbox" className="text-sm cursor-pointer py-1 select-none">
                Use Shoulder Blink Effect
              </label>
            </div>

            {/* ========== headline ========== */}
            <div>
              <Input
                defaultValue={defaultData?.story.title}
                {...register("headline", { required: "A headline is required" })}
                label="Headline*"
              />
              <p className="text-xs p-1 font-light">{errors.headline && errors.headline.message}</p>
            </div>

            {/* ======== alt headline ======== */}
            <Input
              defaultValue={defaultData?.story.meta.altheadline}
              {...register("altheadline")}
              label="Alternative Headline"
            />

            {/* ========== authors ========== */}
            <div>
              <MultiselectSearch
                label="Authors*"
                items={authorsData?.data}
                onSearch={(s) => debounceSearch(() => setSearch((cur) => ({ ...cur, authors: s })))}
                itemsSelected={(i) => {
                  setSelectedAuthors(i as never);
                  setError((cur) => ({ ...cur, authors: "" }));
                }}
                defaultValue={defaultData?.story.authors}
              />
              <p className="text-xs p-1 font-light">{error.authors}</p>
            </div>

            {/* ========== stand first (intro) ========= */}
            <div>
              <Textarea
                defaultValue={defaultData && defaultData?.story.meta.intro}
                {...register("standfirst", { required: "A stand first is required" })}
                label="Stand First*"
                className="focus:ring-0"
              />
              <p className="text-xs p-1 font-light">{errors.standfirst && errors.standfirst.message}</p>
            </div>

            {/* ============= choosing between featured image and featured video ========== */}
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
              <div className="flex flex-col gap-y-4">
                <Radio
                  label="Featured Image"
                  value="image"
                  checked={featuredElement === "image"}
                  onChange={handleSelectFeaturedElement}
                />

                {/* ========== media browser for featured image ======= */}
                <div className="flex flex-col gap-y-3">
                  <div>
                    <Button
                      variant="gradient"
                      onClick={() => {
                        dispatch({ type: "setDialogOpen" });
                        dispatch({ type: "setOpenFrom", payload: "featuredImage" });
                      }}
                    >
                      Featured Image
                    </Button>
                  </div>
                  <div className="max-w-[500px] mt-3">
                    {featuredImgUrl && <img src={`${getImageUrl(featuredImgUrl)}`} alt="Featured_Image" />}
                  </div>
                </div>
                <p className="text-xs p-1 font-light">{error.featuredImage}</p>
              </div>

              {/* ============ featured video ================ */}
              <div className="flex flex-col gap-y-3">
                <Radio
                  label="Featured Embeded Video"
                  value="video"
                  checked={featuredElement === "video"}
                  onChange={handleSelectFeaturedElement}
                />
                <Input
                  defaultValue={defaultData?.story.meta.featured_video}
                  onChange={(e) => {
                    setFeaturedVideo(e.target.value);
                    setError((cur) => ({ ...cur, featuredVideo: "" }));
                  }}
                  label="Featured Embeded Video"
                />
                <div className="mt-3">
                  {featuredVideo && <div dangerouslySetInnerHTML={{ __html: featuredVideo }}></div>}
                </div>
                <p className="text-xs p-1 font-light">{error.featuredVideo}</p>
              </div>
            </div>

            {/* ======== text editor (body) ========= */}
            <div className="mt-6">
              <TinymceEditor
                label="Body*"
                dispatch={dispatch}
                onOpenEmbed={setIsOpenEmbed}
                onOpenEmbedLink={setIsOpenEmbedLink}
                onChange={(content) => {
                  setBody(content);
                }}
                defaultValue={defaultData && defaultData.story.content}
              />
            </div>

            {/* ========== tags ========== */}
            <div>
              <MultiselectSearch
                label="Tags*"
                items={tagsData?.data}
                onSearch={(s) => debounceSearch(() => setSearch((cur) => ({ ...cur, tags: s })))}
                itemsSelected={(i) => {
                  setSelectedTags(i as never);
                  setError((cur) => ({ ...cur, tags: "" }));
                }}
                onAddItem={handleAddTag}
                defaultValue={defaultData && defaultData.story.tags}
              />
              <p className="text-xs p-1 font-light">{error.tags}</p>
            </div>

            {/* ========== categories ========== */}
            <div>
              <MultiselectSearch
                label="Categories*"
                items={categoriesData?.data}
                onSearch={(s) => debounceSearch(() => setSearch((cur) => ({ ...cur, categories: s })))}
                itemsSelected={(i) => {
                  setSelectedCategories(i as never);
                  setError((cur) => ({ ...cur, categories: "" }));
                }}
                defaultValue={defaultData && defaultData.story.categories}
              />
              <p className="text-xs p-1 font-light">{error.categories}</p>
            </div>

            {/* ========== media browser for more images ======= */}
            <div>
              <Button
                variant="gradient"
                onClick={() => {
                  dispatch({ type: "setDialogOpen" });
                  dispatch({ type: "setOpenFrom", payload: "moreImages" });
                }}
              >
                More Images
              </Button>
            </div>
            <div className="grid sm:grid-cols-2 grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-3">
              {moreImages.map((item: { imageUrl: string; imageCaption: string }, index: number) => (
                <div className="relative flex flex-col gap-y-2" key={index}>
                  <img
                    className="w-full rounded-md aspect-video object-cover"
                    src={getImageUrl(item.imageUrl, 320, 180, 40)}
                    alt={item.imageUrl}
                  />
                  <Input
                    onChange={(e) =>
                      setMoreImages((cur: { imageCaption: string }[]) => {
                        cur[index].imageCaption = e.target.value;
                        return cur;
                      })
                    }
                    type="text"
                    label="Caption"
                    defaultValue={item.imageCaption}
                  />
                  <div className="absolute bg-white/75 rounded-lg top-2 right-2">
                    <Icon onClick={() => setMoreImages((cur) => cur.filter((_, idx) => idx !== index))} name="trash" />
                  </div>
                </div>
              ))}
              {moreImages?.length ? (
                <div
                  onClick={() => {
                    dispatch({ type: "setDialogOpen" });
                    dispatch({ type: "setOpenFrom", payload: "moreImages" });
                  }}
                  className="w-full rounded-md cursor-pointer aspect-video bg-gray-300 flex flex-col gap-y-2 items-center justify-center"
                >
                  <span className="text-4xl text-gray-500 rounded-full border-2 border-gray-500 aspect-square w-12 flex items-center justify-center">
                    +
                  </span>
                  <span className="text-gray-500">Add More</span>
                </div>
              ) : null}
            </div>

            {/* =========== media browser =========== */}
            <MediaBrowser
              defaultData={defaultData}
              register={register}
              state={state}
              dispatch={dispatch}
              onFeaturedImgUrl={setFeaturedImgUrl}
              onMoreImgsUrl={setMoreImages}
              onError={setError}
            />

            {/* ============== modal for related news embed ============ */}
            <EmbedRelatedNews open={isOpenEmbed} onOpen={setIsOpenEmbed} />

            {/* ============== modal for link embed ============ */}
            <EmbedLink open={isOpenEmbedLink} onOpen={setIsOpenEmbedLink} />
          </CardBody>
        </HCF.Content>
        <HCF.Footer className="flex w-full px-3 flex-row justify-end">
          <Button type="button" variant="outlined" onClick={() => navigate("/stories")}>
            Cancel
          </Button>
          <Button loading={loading} type="submit">
            {btnLabel}
          </Button>
        </HCF.Footer>
      </HCF>
    </form>
  );
}
